使用Windows的小伙伴如果需要用到Redis, ES或者模拟多个微服务, 会比较麻烦, 下面我们以Redis为例, 看下环境搭建到最终用代码访问到端口的整个过程
- 环境准备
1. 运行 -> control -> 程序和功能 -> 启用或关闭Windows功能 -> 适用于Windows的Linux子系统
2. 从 github 下载 [Windows Terminal](https://github.com/microsoft/terminal/releases) , 使用 powershell 安装
```shell
add-appxpackage ./Microsoft.WindowsTerminal_<版本号>.msixbundle
````
3. 运行 -> cmd
```shell
wsl --status
````
- 如果没有得到响应：
    1. 运行 -> SystemPropertiesAdvanced
    2. 性能 -> 设置 -> 高级选项卡 -> 高级选项卡 -> 虚拟内存 -> 更改
    3. 取消勾选 "自动管理所有驱动器的分页文件大小"
    4. 所有驱动器选择 "系统管理的大小"
    5. 保存后重启Windows
4. 确保WSL版本为2, 否则Docker的守护线程将无法运行
```shell
wsl --set-default-verison 2
```
5. 查看所有Linux发行版
```shell
wsl --list --online

NAME                                   FRIENDLY NAME
Ubuntu                                 Ubuntu
Debian                                 Debian GNU/Linux
kali-linux                             Kali Linux Rolling
Ubuntu-18.04                           Ubuntu 18.04 LTS
Ubuntu-20.04                           Ubuntu 20.04 LTS
Ubuntu-22.04                           Ubuntu 22.04 LTS
OracleLinux_7_9                        Oracle Linux 7.9
OracleLinux_8_7                        Oracle Linux 8.7
OracleLinux_9_1                        Oracle Linux 9.1
openSUSE-Leap-15.5                     openSUSE Leap 15.5
SUSE-Linux-Enterprise-Server-15-SP4    SUSE Linux Enterprise Server 15 SP4
SUSE-Linux-Enterprise-Server-15-SP5    SUSE Linux Enterprise Server 15 SP5
openSUSE-Tumbleweed                    openSUSE Tumbleweed
```
6. 我们根据"NAME"安装第一个Ubuntu
```shell
wsl --install -d Ubuntu
```
7. 查看所有发行版
```shell
wsl -l -v

  NAME      STATE           VERSION
* Ubuntu    Running         2
```
8. 如果安装多个发行版, 可设置默认
```shell
wsl --set-default Ubuntu
```
9. 打开Windows Termial, 点击最上方的下拉按钮, 选择我们刚才安装的Ubuntu, 初次进入需要设定用户名和密码
10. 查看Ubuntu版本
```shell

```
- 进入Linux环境后, 开始安装docker, docker官方已经整理好脚本, 我们用脚本安装就行
1. 安装curl
```shell
sudo apt-get install -y curl
```
2. 使用curl获取安装docker脚本, 等待脚本执行完成
```shell
curl -sSl https://get.docker.com/|sudo sh
```
3. 查看docker版本:
```shell
sudo docker version
```
4. 启动docker服务
```shell
sudo service docker start
```
- 如果出现错误:   
```shell
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
执行这个命令后, 选择1, 再次启动docker服务就可以了
```shell
sudo update-alternatives --config iptables
```
5. 将当前用户加入docker用户组, 这样下面的命令就不用加sudo了
```shell
sudo groupadd docker
sudo gpasswd -a <用户名> docker
```
6. 重启docker服务
```shell
sudo service docker restart
```
- 安装Redis镜像, 运行1个容器实例
1. 拉取[Redis](https://hub.docker.com/_/redis)镜像, 默认latest版本
```shell
docker pull redis
```
2. 查看安装的所有镜像
```shell
docker images
```
3. 运行Redis容器实例, 这里我们叫redis-1, 注意要通过-p参数将端口号6379暴露到外部的6389, 这样Windows进程才能访问到
```shell
docker run --name redis-1 -p 6389:6379 -d redis
```
4. 查看所有容器, 包括已终止的
```shell
docker -a
```
5. 在WSL内部访问Redis
```shell
docker exec -it redis-1 redis-cli
```
6. 进入redis-cli后, 选择数据库0
```shell
select 0
```
7. 插入一条数据
```shell
set mykey abc123
```
8. 查看插入的值
```shell
get mykey
```
- 使用Spring Boot访问Redis, 我用的是JDK1.8, starter版本2.7.11-SNAPSHOT
1. pom中添加配置
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
2. application.properties添加redis的配置, 注意端口号是上文中我们暴露到外部的6389
```properties
spring.redis.host=127.0.0.1
spring.redis.port=6389
spring.redis.database=0
```
3. 测试代码使用以命令行方式运行, 使用RedisAutoConfigeration自动配置
```java
@SpringBootApplication
public class DemoApplication implements CommandLineRunner {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        Object value = redisTemplate.opsForValue().get("mykey");
        System.out.println(value);
}

@Autowired
private  RedisTemplate redisTemplate;
```
- 这里发现value为null, 经查发现是序列化相关问题
```java
redisTemplate.opsForValue().set("mykey2", "qwer123");
```
插入这条数据后我们到WSL里的redis-cli查询所有的key
```shell
keys *

1) "\xac\xed\x00\x05t\x00\x06mykey2"
2) "mykey"
```
发现刚插入的mykey2前面多了一段乱码"\xac\xed\x00\x05t\x00\x06", 通过指定RedisTemplate的泛型可以解决这个问题
```java
@Autowired
private  RedisTemplate<String, String> redisTemplate;
```


