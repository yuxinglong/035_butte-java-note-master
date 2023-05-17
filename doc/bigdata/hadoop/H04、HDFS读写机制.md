# 一、读写机制

## 1、数据写入

![](https://images.gitee.com/uploads/images/2022/0213/131948_1c43bbc4_5064118.png "05-1.png")

- 客户端访问NameNode请求上传文件；
- NameNode检查目标文件和目录是否已经存在；
- NameNode响应客户端是否可以上传；
- 客户端请求NameNode文件块Block01上传服务位置；
- NameNode响应返回3个DataNode节点；
- 客户端通过输入流建立DataNode01传输通道；
- DataNode01调用DataNode02，DataNode02调用DataNode03，通信管道建立完成；
- DataNode01、DataNode02、DataNode03逐级应答客户端。
- 客户端向DataNode01上传第一个文件块Block；
- DataNode01接收后传给DataNode02，DataNode02传给DataNode03;
- Block01传输完成之后，客户端再次请求NameNode上传第二个文件块；

## 2、数据读取

![](https://images.gitee.com/uploads/images/2022/0213/132003_4a89ae96_5064118.png "05-2.png")

- 客户端通过向NameNode请求下载文件；
- NameNode查询获取文件元数据并返回；
- 客户端通过元数据信息获取文件DataNode地址；
- 就近原则选择一台DataNode服务器，请求读取数据；
- DataNode传输数据返回给客户端；
- 客户端以本地处理目标文件；

# 二、基础API案例

## 1、基础演示接口

```java
public interface HdfsFileService {

    // 创建文件夹
    void mkdirs(String path) throws Exception ;

    // 文件判断
    void isFile(String path) throws Exception ;

    // 修改文件名
    void reName(String oldFile, String newFile) throws Exception ;

    // 文件详情
    void fileDetail(String path) throws Exception ;

    // 文件上传
    void copyFromLocalFile(String local, String path) throws Exception ;

    // 拷贝到本地：下载
    void copyToLocalFile(String src, String dst) throws Exception ;

    // 删除文件夹
    void delete(String path) throws Exception ;

    // IO流上传
    void ioUpload(String path, String local) throws Exception ;

    // IO流下载
    void ioDown(String path, String local) throws Exception ;

    // 分块下载
    void blockDown(String path, String local1, String local2) throws Exception ;
}
```

## 2、命令API用法

```java
@Service
public class HdfsFileServiceImpl implements HdfsFileService {

    @Resource
    private HdfsConfig hdfsConfig ;

    @Override
    public void mkdirs(String path) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                               configuration, "root");
        // 2、创建目录
        fileSystem.mkdirs(new Path(path));
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void isFile(String path) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                               configuration, "root");
        // 2、判断文件和文件夹
        FileStatus[] fileStatuses = fileSystem.listStatus(new Path(path));
        for (FileStatus fileStatus : fileStatuses) {
            if (fileStatus.isFile()) {
                System.out.println("文件:"+fileStatus.getPath().getName());
            }else {
                System.out.println("文件夹:"+fileStatus.getPath().getName());
            }
        }
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void reName(String oldFile, String newFile) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                configuration, "root");
        // 2、修改文件名
        fileSystem.rename(new Path(oldFile), new Path(newFile));
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void fileDetail(String path) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                               configuration, "root");
        // 2、读取文件详情
        RemoteIterator<LocatedFileStatus> listFiles =
                                    fileSystem.listFiles(new Path(path), true);
        while(listFiles.hasNext()){
            LocatedFileStatus status = listFiles.next();
            System.out.println("文件名："+status.getPath().getName());
            System.out.println("文件长度："+status.getLen());
            System.out.println("文件权限："+status.getPermission());
            System.out.println("所属分组："+status.getGroup());
            // 存储块信息
            BlockLocation[] blockLocations = status.getBlockLocations();
            for (BlockLocation blockLocation : blockLocations) {
                // 块存储的主机节点
                String[] hosts = blockLocation.getHosts();
                for (String host : hosts) {
                    System.out.print(host+";");
                }
            }
            System.out.println("==============Next==============");
        }
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void copyFromLocalFile(String local, String path) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                configuration, "root");
        // 2、执行上传操作
        fileSystem.copyFromLocalFile(new Path(local), new Path(path));
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void copyToLocalFile(String src,String dst) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                               configuration, "root");
        // 2、执行下载操作
        // src 服务器文件路径 ; dst 文件下载到的路径
        fileSystem.copyToLocalFile(false, new Path(src), new Path(dst), true);
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void delete(String path) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                configuration, "root");
        // 2、删除文件或目录 是否递归
        fileSystem.delete(new Path(path), true);
        // 3、关闭资源
        fileSystem.close();
    }

    @Override
    public void ioUpload(String path, String local) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                configuration, "root");
        // 2、输入输出流
        FileInputStream fis = new FileInputStream(new File(local));
        FSDataOutputStream fos = fileSystem.create(new Path(path));
        // 3、流对拷
        IOUtils.copyBytes(fis, fos, configuration);
        // 4、关闭资源
        IOUtils.closeStream(fos);
        IOUtils.closeStream(fis);
        fileSystem.close();
    }

    @Override
    public void ioDown(String path, String local) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                configuration, "root");
        // 2、输入输出流
        FSDataInputStream fis = fileSystem.open(new Path(path));
        FileOutputStream fos = new FileOutputStream(new File(local));
        // 3、流对拷
        IOUtils.copyBytes(fis, fos, configuration);
        // 4、关闭资源
        IOUtils.closeStream(fos);
        IOUtils.closeStream(fis);
        fileSystem.close();
    }

    @Override
    public void blockDown(String path,String local1,String local2) throws Exception {
        readFileSeek01(path,local1);
        readFileSeek02(path,local2);
    }

    private void readFileSeek01(String path,String local) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                                configuration, "root");
        // 2、输入输出流
        FSDataInputStream fis = fileSystem.open(new Path(path));
        FileOutputStream fos = new FileOutputStream(new File(local));
        // 3、部分拷贝
        byte[] buf = new byte[1024];
        for(int i =0 ; i < 1024 * 128; i++){
            fis.read(buf);
            fos.write(buf);
        }
        // 4、关闭资源
        IOUtils.closeStream(fos);
        IOUtils.closeStream(fis);
        fileSystem.close();
    }

    private void readFileSeek02(String path,String local) throws Exception {
        // 1、获取文件系统
        Configuration configuration = new Configuration();
        FileSystem fileSystem = FileSystem.get(new URI(hdfsConfig.getNameNode()),
                configuration, "root");
        // 2、输入输出流
        FSDataInputStream fis = fileSystem.open(new Path(path));
        // 定位输入数据位置
        fis.seek(1024*1024*128);
        FileOutputStream fos = new FileOutputStream(new File(local));
        // 3、流拷贝
        IOUtils.copyBytes(fis, fos, configuration);
        // 4、关闭资源
        IOUtils.closeStream(fos);
        IOUtils.closeStream(fis);
        fileSystem.close();
    }
}
```

## 3、合并切割文件

```
cat hadoop-2.7.2.zip.block1 hadoop-2.7.2.zip.block2 > hadoop.zip
```

# 三、机架感知

Hadoop2.7的文档说明

![](https://images.gitee.com/uploads/images/2022/0213/132027_c3fb9786_5064118.png "05-3.png")

第一个副本和client在一个节点里，如果client不在集群范围内，则这第一个node是随机选取的；第二个副本和第一个副本放在相同的机架上随机选择；第三个副本在不同的机架上随机选择，减少了机架间的写流量，通常可以提高写性能,机架故障的概率远小于节点故障的概率，因此该策略不会影响数据的稳定性。

# 四、网络拓扑

HDFS写数据的过程中，NameNode会选择距离待上传数据最近距离的DataNode接收数据，基于机架感知，NameNode就可以画出上图所示的datanode网络拓扑图。D1,R1都是交换机，最底层是datanode。

![](https://images.gitee.com/uploads/images/2022/0213/132044_5370838e_5064118.png "05-4.png")

```
Distance(/D1/R1/N1,/D1/R1/N1)=0  相同的节点
Distance(/D1/R1/N1,/D1/R1/N2)=2  同一机架下的不同节点
Distance(/D1/R1/N1,/D1/R2/N1)=4  同一IDC下的不同datanode
Distance(/D1/R1/N1,/D2/R3/N1)=6  不同IDC下的datanode
```

**源码参考：** https://gitee.com/cicadasmile/big-data-parent/tree/master/series01-hadoop-parent