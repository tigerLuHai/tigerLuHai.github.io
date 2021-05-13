---
title: Hadoop之HDFS上传文件源码分析
top: false
cover: false
toc: true
mathjax: true
date: 2020-01-16 21:16:23
password:
summary:
tags:
categories: 大数据
---



## 概述

hdfs中每个block默认情况下是128M，由于每个块比较大，所以在写数据的过程中是把数据块拆分成一个个的数据包以管道的形式发送的，所以hdfs文件的写入会涉及到客户端、namenode、datanode多个模块的交互。

### 操作代码

```java
Configuration conf = new Configuration();  
FileSystem fs = FileSystem.get(conf);  
Path file = new Path("hdfs://127.0.0.1:9000/example.txt");  
FSDataOutputStream outStream = fs.create(file);  
out.write("java api write data".getBytes("UTF-8"));   
outStream.close();  
```


通过 FileSystem.get(conf); 来构造了一个FileSystem 实例，这里对应的是DistributedFileSystem，通过调用DistributedFileSystem里面的create方法创建了一个文件，并且返回了这个文件的输出流，用于写入数据。

DistributedFileSystem的create方法有很多重载的方法，最终调用了DistributedFileSystem的下面的这个create方法

```java
@Override
public FSDataOutputStream create(final Path f, final FsPermission permission,
    final EnumSet<CreateFlag> cflags, final int bufferSize,
    final short replication, final long blockSize, final Progressable progress,
    final ChecksumOpt checksumOpt) throws IOException {
    statistics.incrementWriteOps(1);
    Path absF = fixRelativePart(f);
    return new FileSystemLinkResolver<FSDataOutputStream>() {
      @Override
      public FSDataOutputStream doCall(final Path p)throws IOException, UnresolvedLinkException {
        final DFSOutputStream dfsos = dfs.create(getPathName(p), permission,
                cflags, replication, blockSize, progress, bufferSize,
                checksumOpt);
        return dfs.createWrappedOutputStream(dfsos, statistics);

 }

      @Override
      public FSDataOutputStream next(final FileSystem fs, final Path p) throws IOException {
        return fs.create(p, permission, cflags, bufferSize,replication, blockSize, progress, checksumOpt);
      }
		}.resolve(this, absF);
  }

```


在这里，调用了DFSClient的create方法来创建文件

```java
create(String, FsPermission, EnumSet, boolean, short, long, Progressable, int, ChecksumOpt, InetSocketAddress[])
```

在这里create方法里，通过DFSOutputStream的静态方法newStreamForCreate构建了一个对象，并且返回了一个DFSOutputStream对象。

```java
static DFSOutputStream newStreamForCreate(DFSClient dfsClient, String src,
      FsPermission masked, EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize, Progressable progress, int buffersize,
      DataChecksum checksum, String[] favoredNodes) throws IOException {
    TraceScope scope = dfsClient.getPathTraceScope("newStreamForCreate", src);
    try {
      HdfsFileStatus stat = null;
      // Retry the create if we get a RetryStartFileException up to a maximum
      // number of times
      boolean shouldRetry = true;
      int retryCount = CREATE_RETRY_COUNT;
      while (shouldRetry) {
        shouldRetry = false;
        try {
          //通过调用namenode的create方法来创建文件
          stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
              new EnumSetWritable<CreateFlag>(flag), createParent, replication,
              blockSize, SUPPORTED_CRYPTO_VERSIONS);
          break;
        } catch (RemoteException re) {
          IOException e = re.unwrapRemoteException(
              AccessControlException.class,
              DSQuotaExceededException.class,
              FileAlreadyExistsException.class,
              FileNotFoundException.class,
              ParentNotDirectoryException.class,
              NSQuotaExceededException.class,
              RetryStartFileException.class,
              SafeModeException.class,
              UnresolvedPathException.class,
              SnapshotAccessControlException.class,
              UnknownCryptoProtocolVersionException.class);
          if (e instanceof RetryStartFileException) {
            if (retryCount > 0) {
              shouldRetry = true;
              retryCount--;
            } else {
              throw new IOException("Too many retries because of encryption" +
                  " zone operations", e);
            }
          } else {
            throw e;
          }
        }
      }
  Preconditions.checkNotNull(stat, "HdfsFileStatus should not be null!");

  //构造了一个DFSOutputStream对象，即刚刚创建的文件的输出流.
  final DFSOutputStream out = new DFSOutputStream(dfsClient, src, stat,
      flag, progress, checksum, favoredNodes);

 //start方法启动了DFSOutputStream的内部类DataStreamer，用于接收要写入的数据包
  out.start();
  return out;
} finally {
  scope.close();
  }
}
```

通过dfsClient.namenode.create在hdfs的目录树上创建了一个文件，然后通过new DFSOutputStream创建了一个该文件的输出流实例，在DFSOutputStream构造方法中,初始化了用于数据处理的DFSOutputStream类的内部类DataStreamer，用于启动DataStreamer线程,接受客户端写入数据包的请求。

DataStreamer是一个线程，它的启动是通过DFSOutputStream的start方法来启动的

```java
 /** Construct a new output stream for creating a file. */
  private DFSOutputStream(DFSClient dfsClient, String src, HdfsFileStatus stat,
      EnumSet<CreateFlag> flag, Progressable progress,
      DataChecksum checksum, String[] favoredNodes) throws IOException {
      this(dfsClient, src, progress, stat, checksum);
      this.shouldSyncBlock = flag.contains(CreateFlag.SYNC_BLOCK);
      computePacketChunkSize(dfsClient.getConf().writePacketSize, bytesPerChecksum);

      streamer = new DataStreamer(stat, null);
      if (favoredNodes != null && favoredNodes.length != 0) {
      	streamer.setFavoredNodes(favoredNodes);
      }
}
```

### namenode创建文件

上述dfsClient.namenode.create是调用了客户端和namenode交互的接口ClientProtocol中的create方法来创建文件，之后由ClientProtoco的实现类
org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB中的create方法封装了创建文件所需的信息，通过rpc的方式发送到了namenode来处理。

最终的实现方法是NameNodeRpcServer类的create方法，之后经过FSNamesystem的startFile、startFileInt，最后在方法startFileInternal中实现具体的逻辑。

1.首先检查是否是一个目录，如果是的话抛出异常.
2.检查是否有写的权限。
3.检查是否创建父目录
4.检查create字段，用户是否创建为文件
5.检查是否覆盖源文件，如果true的话，则删除原来的旧文件。

最后调用了FSDirectory的addFile方法来创建文件。

```java
iip = dir.addFile(parent.getKey(), parent.getValue(), permissions,

replication, blockSize, holder, clientMachine);
```

具体的操作就是找到该文件的父目录，然后在父目录的List类型的对象children中添加一条数据。

具体的代码如下：

```java
 private BlocksMapUpdateInfo startFileInternal(FSPermissionChecker pc, 
 	INodesInPath iip, PermissionStatus permissions, String holder,
 	String clientMachine, boolean create, boolean overwrite, 
 	boolean createParent, short replication, long blockSize, 
 	boolean isLazyPersist, CipherSuite suite, CryptoProtocolVersion version,
 	EncryptedKeyVersion edek, boolean logRetryEntry)throws IOException {
    assert hasWriteLock();
    // Verify that the destination does not exist as a directory already.
    final INode inode = iip.getLastINode();
    final String src = iip.getPath();
    //检查是否存在，并且是不是一个目录
    if (inode != null && inode.isDirectory()) {
      throw new FileAlreadyExistsException(src +
          " already exists as a directory");
    }

    //检查是否有写的权限
    final INodeFile myFile = INodeFile.valueOf(inode, src, true);
    if (isPermissionEnabled) {
      if (overwrite && myFile != null) {
        dir.checkPathAccess(pc, iip, FsAction.WRITE);
      }
      /*
       * To overwrite existing file, need to check 'w' permission 
       * of parent (equals to ancestor in this case)
       */
      dir.checkAncestorAccess(pc, iip, FsAction.WRITE);
    }
    //是否创建父目录
    if (!createParent) {
      dir.verifyParentDir(iip, src);
    }

    FileEncryptionInfo feInfo = null;

    final EncryptionZone zone = dir.getEZForPath(iip);
    if (zone != null) {
      // The path is now within an EZ, but we're missing encryption parameters
      if (suite == null || edek == null) {
        throw new RetryStartFileException();
      }
      // Path is within an EZ and we have provided encryption parameters.
      // Make sure that the generated EDEK matches the settings of the EZ.
      final String ezKeyName = zone.getKeyName();
      if (!ezKeyName.equals(edek.getEncryptionKeyName())) {
        throw new RetryStartFileException();
      }
      feInfo = new FileEncryptionInfo(suite, version,
          edek.getEncryptedKeyVersion().getMaterial(),
          edek.getEncryptedKeyIv(),
          ezKeyName, edek.getEncryptionKeyVersionName());
    }

    try {
      BlocksMapUpdateInfo toRemoveBlocks = null;
      if (myFile == null) {
      //是否创建文件
        if (!create) {
          throw new FileNotFoundException("Can't overwrite non-existent " +
              src + " for client " + clientMachine);
        }
      } else {
         //是否覆盖
        if (overwrite) {
          toRemoveBlocks = new BlocksMapUpdateInfo();
          List<INode> toRemoveINodes = new ChunkedArrayList<INode>();
          //删除旧文件
          long ret = FSDirDeleteOp.delete(dir, iip, toRemoveBlocks,
                                          toRemoveINodes, now());
          if (ret >= 0) {
            iip = INodesInPath.replace(iip, iip.length() - 1, null);
            FSDirDeleteOp.incrDeletedFileCount(ret);
            removeLeasesAndINodes(src, toRemoveINodes, true);
          }
        } else {
          // If lease soft limit time is expired, recover the lease
          recoverLeaseInternal(RecoverLeaseOp.CREATE_FILE,
              iip, src, holder, clientMachine, false);
          throw new FileAlreadyExistsException(src + " for client " +
              clientMachine + " already exists");
        }
      }

      checkFsObjectLimit();
      INodeFile newNode = null;

      // Always do an implicit mkdirs for parent directory tree.
      Map.Entry<INodesInPath, String> parent = FSDirMkdirOp
          .createAncestorDirectories(dir, iip, permissions);
      if (parent != null) {
        //具体的操作，创建文件
        iip = dir.addFile(parent.getKey(), parent.getValue(), permissions,
            replication, blockSize, holder, clientMachine);
        newNode = iip != null ? iip.getLastINode().asFile() : null;
      }

      if (newNode == null) {
        throw new IOException("Unable to add " + src +  " to namespace");
      }
      leaseManager.addLease(newNode.getFileUnderConstructionFeature()
          .getClientName(), src);

      // Set encryption attributes if necessary
      if (feInfo != null) {
        dir.setFileEncryptionInfo(src, feInfo);
        newNode = dir.getInode(newNode.getId()).asFile();
      }

      setNewINodeStoragePolicy(newNode, iip, isLazyPersist);

      // record file record in log, record new generation stamp
      getEditLog().logOpenFile(src, newNode, overwrite, logRetryEntry);
      NameNode.stateChangeLog.debug("DIR* NameSystem.startFile: added {}" +
          " inode {} holder {}", src, newNode.getId(), holder);
      return toRemoveBlocks;
    } catch (IOException ie) {
      NameNode.stateChangeLog.warn("DIR* NameSystem.startFile: " + src + " " +
          ie.getMessage());
      throw ie;
    }
```
