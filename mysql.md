# 使用Glances实时监控电脑各项性能数据并将数据输出到MySql
## 1.1 Glances简介
Glances 是一个由 Python 编写，使用 psutil 库来从系统抓取信息的基于 curses 开发的跨平台命令行系统监视工具。 通过 Glances，我们可以监视 CPU，平均负载，内存，网络流量，磁盘 I/O，其他处理器 和 文件系统 空间的利用情况。
## 1.2 Glances的安装(在Ubuntu16.04)
在命令行输入:

    llw@llw-Dell-Precision-M3800:~$ wget -O- https://bit.ly/glances | /bin/bash
安装完成后,在命令行输入:

    llw@llw-Dell-Precision-M3800:~$ glances

然后会得到下面这样的输出,就表明安装成功,并可以开始检测自己的电脑了:

![image](https://github.com/leondelee/photos/blob/master/glances.png)

## 1.3 将Glances数据上传到网页
在命令行输入:

    llw@llw-Dell-Precision-M3800:~$ glances -w
会得到下面输出:

    Glances web server started on http://0.0.0.0:61208/
这表明我们可以通过"http://0.0.0.0:61208/"这个地址在网页上访问到glances监控得到的数据.在浏览器输入这个地址,我们得到了这样子的结果:

![glances_web](https://github.com/leondelee/photos/blob/master/glances_web.png)

# 2.将Glances得到的数据导入到Mysql当中
## 2.1 在Mysql当中创建数据库和表
### 2.1.1 使用APT安装mysql
首先去官网下载Mysql仓库, 地址是[http://dev.mysql.com/downloads/repo/apt/]( http://dev.mysql.com/downloads/repo/apt/),当然要根据自己的操作系统选择合适的版本.我选择了Ubuntu - 16.04 LTS.
然后解压缩:

    sudo dpkg -i /PATH/version-specific-package-name.deb

更新源:

    sudo apt-get update

安装:

    sudo apt-get install mysql-server

需要注意的是,这里虽然安装的是mysql-server, 但是也会安装client, 所以大家不用担心. 
Mysql会在安装完成以后自动启动,因此我们可以使用

    sudo service mysql status
来验证是否安装成功,并查看mysql服务器的工作状态.

### 2.1.2 创建数据库并创建表
关于Mysql的用法,就不在这里赘述了.
首先创建数据库ubuntu_monitor,数据都将储存在这个数据库中:

    mysql> create database ubuntu_monitor
随后,创建储存数据的表, 一共有9列,分别是:Time, Os, Mem_tatol, Mem_avail, Mem_used, Mem_per, Cpu_core, Cpu_per, Cpu_temp,分别对应: 写入时间, 操作系统名称,总内存, 可用内存, 已用内存, 内存使用率, cpu核数, cpu使用率, cpu温度.

    mysql> create table $table_name(
        Time VARCHAR(100),
        Os VARCHAR(40),
        Mem_total VARCHAR(40),
        Mem_avail VARCHAR(40),
        Mem_used VARCHAR(40),
        Mem_per VARCHAR(40),
        Cpu_core VARCHAR(40),
        Cpu_per VARCHAR(40),
        Cpu_temp VARCHAR(40)
    );

## 2.2 读取Glances的网页数据
到了现在,我们已经做好储存数据的一切准备,只需要读取网页数据就可以了. 编写php脚本完成这一工作.注意, [glances官方文档](https://github.com/nicolargo/glances/wiki/The-Glances-RESTFULL-JSON-API)当中有httpAPI的详细配制方法, 可以参考一下. 我的脚本名字是test.php,要储存在/var/www/html/目录下.脚本如下:


    <?php
        $mysql_server_name = 'localhost'; //改成自己的mysql数据库服务器
        $mysql_port = 3306;
        $mysql_username = 'root'; //改成自己的mysql数据库用户名
        $mysql_password = 'passwd'; //改成自己的mysql数据库密码 
        $mysql_database = 'dbname'; //改成自己的mysql数据库名

        $conn = new mysqli($mysql_server_name,$mysql_username,$mysql_password,$mysql_database);

        if (!$conn){
        die('Could not connect: ' . mysql_error());
        } 

        echo "Connected Successfully!<br>";
        $table_name = "my_monitor";
        // 使用 sql 创建数据表
        $sql = "create table $table_name(
        Time VARCHAR(100),
        Os VARCHAR(40),
        Mem_total VARCHAR(40),
        Mem_avail VARCHAR(40),
        Mem_used VARCHAR(40),
        Mem_per VARCHAR(40),
        Cpu_core VARCHAR(40),
        Cpu_per VARCHAR(40),
        Cpu_temp VARCHAR(40)
        )";

        if ($conn->query($sql) === TRUE) {
            echo "Table  created successfully<br>";
        } else {
            echo "创建数据表错误: " . $conn->error."<br>";
        }
        // 每5秒读一次数据,并写入数据库
        while (true) {
        echo "*************************************************<br>";
        $all_data = json_decode(file_get_contents('http://0.0.0.0:61208/api/2/all'));  
        $os = $all_data->{'system'}->{'linux_distro'};
        $mem_total = $all_data->{'mem'}->{'total'};
        $mem_avail = $all_data->{'mem'}->{'available'}; 
        $mem_used = $all_data->{'mem'}->{'used'};
        $mem_per = $all_data->{'mem'}->{'percent'};
        $cpu_core = $all_data->{'cpu'}->{'cpucore'};
        $cpu_per = $all_data->{'cpu'}->{'total'};
        $cpu_temp = substr(file_get_contents('http://0.0.0.0:61208/api/2/sensors'),-20,2);
        $now = $all_data->{'now'};

        $isrt = "INSERT INTO $table_name(Time,Os,Mem_total,Mem_avail,Mem_used,Mem_per,Cpu_core,Cpu_per,Cpu_temp) VALUES ('$now','$os','$mem_total','$mem_avail','$mem_used','$mem_per','$cpu_core','$cpu_per','$cpu_temp');";

        if ($conn->query($isrt) === True) {
                        echo "Data:Time:$now,Os:$os,Mem_total:$mem_total,Mem_avail:$mem_avail,Mem_used:$mem_used,Mem_per:$mem_per,Cpu_core:$cpu_core,Cpu_per:$cpu_per,Cpu_temp:$cpu_temp inserted successfully!<br>";
        } else {
                        echo "Error".$conn->error;
        }
        sleep(5);
        }
        $conn->close();

        ?>

## 2.3 运行PHP脚本
在命令行输入:

    sudo /etc/init.d/apache2 restart
然后在浏览器当中输入:

    http://localhost/test.php
这样子脚本就运行成功了.
打开数据库验证:
![monitor](https://github.com/leondelee/photos/blob/master/monitor.png)
