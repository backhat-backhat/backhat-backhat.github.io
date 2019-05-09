---
layout: post
title: "时区转换问题"
date: 2017-04-09 23:30:00
tags: redis 时区转换
---
对时区转换问题的总结

最近在码代码的时候，碰到了一个时区转换的问题。不同主机和软件内部时区设置不同，将时间转换成秒数的时候，不同时区转换结果是不同的，如果需要处理，可以按照某一个统一的时区方式进行转换，这样后期处理会方便的多。

# 什么是时区
<p style="font-family:consolas">
A time zone is a region of the globe that observes a uniform standard time for legal, commercial, and social purposes. Time zones tend to follow the boundaries of countries and their subdivisions because it is convenient for areas in close commercial or other communication to keep the same time. (from wikipedia)
</p>
通常世界时区表的表盘上会标示着全球24个时区的城市名称，但究竟这24个时区是如何产生的？过去世界各地原本各自订定当地时间，但随着交通和电讯的发达，各地交流日益频繁，不同的地方时间，造成许多困扰，于是在西元1884年的国际会议上制定了全球性的标准时，明定以英国伦敦格林威治这个地方为零度经线的起点（亦称为本初子午线），并以地球由西向东每24小时自转一周360°，订定每隔经度15°，时差1小时。而每15°的经线则称为该时区的中央经线，将全球划分为24个时区，其中包含23个整时区及180°经线左右两侧的2个半时区。就全球的时间来看，东经的时间比西经要早，也就是如果格林威治时间是中午12时，则中央经线15°E的时区为下午1时，中央经线30°E时区的时间为下午2时；反之，中央经线15°W的时区时间为上午11时，中央经线30°W时区的时间为上午10时。以台湾为例，台湾位于东经121°，换算后与格林威治就有8小时的时差。如果两人同时从格林威治的0°各往东、西方前进，当他们在经线180°时，就会相差24小时，所以经线180°被定为国际换日线，由西向东通过此线时日期要减去一日，反之，若由东向西则要增加一日。
(摘自：https://www.douban.com/note/147740972/)

早期是以 GMT (Greenwich Mean Time，格林威治时间)作为标准时间，但是，由于地球在它的椭圆轨道里的运动速度不均匀，这个时刻可能和实际的太阳时相差16分钟，地球每天的自转是有些不规则的，而且正在缓慢减速。所以，后来使用的是协调世界时(UTC, Coordinated Universal Time)。

查看世界时区表： <br>
1. [list of time zones by country](https://en.wikipedia.org/wiki/List_of_time_zones_by_country) <br>
2. [list of time zones](https://en.wikipedia.org/wiki/Lists_of_time_zones)

# UTC
UTC，即 Coordinated Universal Time，协调世界时，现在都是以这个作为世界时的基准，通过在时间后面直接加上一个大写字母“Z”表示。Z 表示的是 UTC 的 0 偏移(the zero UTC offset)。比如，“09:30 UTC” 可以写成 “09:30Z” 或者 “0930Z”。“14:45:15 UTC” 可写成 “14:45:14Z” 或者 “144515Z”。

Offset from UTC，表示的是当前时区与 UTC 基准时间的偏差。比如中国大陆时间以北京时间为标准，比 UTC 快 8 个小时，记为 UTC+8，称为东 8 区。如果比 UTC 时间慢，比如夏威夷时间比 UTC 慢 10 个小时，记为 UTC-10，称为西 10 区。

# 中国时区(Time in China)
中国时区以 UTC+08:00 作为标准时间(eight hours ahead of Coordinated Universal Time)，即使中国在地理上跨越了 5 个时区，仍然只有这一个时区作为标准。官方称中国标准时间为北京时间(Beijing Time)，也记为 CST (China Standard Time)。

> 注意，CST 这个缩写，很具有二义性。因为很多时区的缩写都记为 CST。

特别行政区(the special administrative regions, SARs)有他们自己的时区，比如 Hong Kong Time (香港时间) 和 Macau Standard Time (澳门标准时间)，不过 1992 年之后，他们都相等于北京时间。

另外，另一个时间标准就是新疆时间，这个时间比北京时间晚两个小时(UTC +06:00)，一般称为乌鲁木齐时间或者新疆时间。

# 时区转换
两个时区 A 和 B 之间的转换关系如下所示: 

	"time in zone A" − "UTC offset for zone A" = "time in zone B" − "UTC offset for zone B"
等式两边都等于 UTC 时间，所以相等。如果将上面的等式变换一下位置，为

	"time in zone A" = "time in zone B" − "UTC offset for zone B" + "UTC offset for zone A"
可以根据其中一个时区 B，计算出对应的时区 A 当前的时间。比如计算纽约时间 09:30 (EST, -05) 时洛杉矶时间(PST, UTC Offset = -08) 的时间：

	洛杉矶时间 = 09:30 - (-05:00) + (-08:00) = 06:30

# Unix Time
大部分类 Unix 系统，比如 Linux 或者 Mac OS X，都是以 UTC 作为系统时间，而不是将计算机设置为某一个特定的时区。标准库函数可通过当前时区计算本地时间，时区一般是通过环境变量 TZ 获取。这允许不同时区的用户，只需要在系统中设置不同的时区，就能够使用同一台计算机正确显示各自不同时区的时间。时区信息一般都来自 IANA 时区数据库，很多系统，包括 GNU C 函数库，都是使用的这个时区数据库。

一般在 Linux 中，如果安装了时区数据库，在 `/usr/share/zoneinfo/` 中可查看。

## linux c 处理时间的函数
1、 `time_t time(time_t * tloc)` <br>
返回的是当前时间距离 Epoch 的秒数。 Epoch 指的特定时间 `1970-01-01 00:00:00 UTC`。

2、 `struct tm* gmtime(const time_t * timep)` <br>
参数是一个 time_t 类型的指针，可以使用上面 time() 的返回值作为参数，返回一个 tm 的结构体指针。

	struct tm {
		int tm_sec;		// 秒数, 0-59
		int tm_min;		//0-59
		int tm_hour;	//0-23
		int tm_mday;	//1-31
		int tm_mon;		//0-11
		int tm_year;	//从 1900 开始的年数，一般计算当前时间的时候，需要加上 1900
		int tm_wday;	//day in the week, 从周日开始，0-6
		int tm_yday;	//day int the year, 从1月1日开始，0-365
		int tm_isdst;	//daylight saving time，标志位，0表示不生效
	};
函数参数是 time_t 类型的指针变量，返回的是 GMT 的时间。

3、 `struct tm* localtime(const time_t* tomep)` <br>
参数与 gmtime() 相同，返回值也是 tm 结构指针，但是返回的是本地时区的时间。 localtime 调用时，相当于调用了 `tzset()` 函数，将当前系统时区设置为全局变量 tzname。返回的时间，经过转换，表示的是当前时区的时间。

4、 `time_t mktime*(struct tm* tm)` <br>
该函数，将tm 结构体所表示的时间转换成本地时间的描述。该函数忽略 tm 结构中的 `tm_wday` 和 `tm_yday` 这两个参数，但是需要根据 `tm_isdst` 判断 `daylight saving time` 是否生效。函数会修改 tm 结构体中 `tm_wday` 和 `tm_yday` 这两个成员，同时保留 `tm_isdst` 成员的属性不变，还会根据当前系统时区信息修改全局变量 tzname 。

5、 `void tzset(void)` <br>
初始化时间转换信息 (initialize time conversion information)

	extern char* tzname[2];
	extern long timezone;
	extern int daylight;
这三个是全局变量，表示的时间相关信息。

tzset() 函数使用系统环境变量 TZ 设置 tzname 全局变量，如果系统没有设置 TZ 环境变量，系统一般会将系统时区目录(system timezone directory)中的 `tzfile-format file` localtime 这个文件的信息对 tzname 进行初始化。该文件一般在 `/etc/localtime`。

如果设置了 TZ 环境变量，但是内容为空，或者内容不能识别为某个时区信息，那么将使用 UTC 作为时区标准。

一般，shell 中的时间命令 date 也是以 TZ 作为时区信息，如果TZ不存在，以系统默认时区信息为准。可以通过修改 /etc/localtime 这个文件的方法达到修改系统时区的目的。

# 修改 linux 系统的时区
如果在系统中安装了时区数据库，一般默认在 `/usr/share/zoneinfo/` 目录下，可以通过修改 /etc/localtime 的方法设置系统的时区信息。

查看当前系统时区信息，date 查看

	Sun Apr 9 10:00:00 EDT 2016
CST 即为时区，如果修改成北京时间，及 UTC +08:00，在时区数据库中，对应的时区格式文件为 `Asia/Shanghai`

	rm /etc/localtime
	ln /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
这样就设置了当前系统的时区，这种方法永久生效。这种方法比较简便。还可以有其他方法，想知道的童鞋可以去查查资料。

# mysql 设置时区的方法
网上有很多小伙伴都说，可以通过在 my.cnf 配置文件中添加参数修改。

	# vi my.cnf，在 [mysqld] 节点下添加
	default-time_zone = '+08:00'
然后重启 mysql 服务器即可修改 mysql 的时区。

查看 mysql 的时区

	mysql> show variables like '%time_zone%';
	mysql> select @@global.time_zone, @@session.time_zone;
当然，如果想临时生效，可以设置 `time_zone` 变量的方法。

	mysql> set time_zone = "+08:00";

---
楼主通过在 mysql 手册中查找，找到了下面这种方法，但是需要系统中安装有时区数据库。

MySQL服务器有几个时区设置：

- 系统时区。服务器启动时便试图确定主机的时区，用它来设置system_time_zone系统变量。

- 服务器当前的时区。全局系统变量time_zone表示服务器当前使用的时区。初使值为'SYSTEM'，说明服务器时区与系统时区相同。可以用--default-time-zone=timez选项显式指定初使值。如果你有SUPER 权限，可以用下面的语句在运行时设置全局变量值：

	mysql> SET GLOBAL time_zone = timezone;
- 每个连接的时区。每个客户端连接有自己的时区设置，用会话 `time_zone` 变量给出。其初使值与全局变量 `time_zone` 相同，但可以用下面的语句重设：

	mysql> SET time_zone = timezone; <br>
可以用下面的方法查询当前的全局变量值和每个连接的时区：

	`mysql> SELECT @@global.time_zone, @@session.time_zone;` <br>
timezone值为字符串，表示UTC的偏移量，例如'+10:00'或'-6:00'。

如果系统中安装有时区数据库，通常是 `/usr/share/zoneinfo`，使用 `mysql_tzinfo_to_sql` 程序装载时区表。

	shell> mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql
`mysql_tzinfo_to_sql`读取系统时区文件并生成SQL语句。mysql处理这些语句并装载时区表。

`mysql_tzinfo_to_sql`还可以用来装载单个时区文件，并生成闰秒信息。

要想装载对应时区`tz_name`的单个时区文件`tz_file`，应这样调用`mysql_tzinfo_to_sql`：

	shell> mysql_tzinfo_to_sql tz_file tz_name | mysql -u root mysql
如果你的时区需要计算闰秒，按下面方法初使化闰秒信息，其中`tz_file`是时区文件名：

	shell> mysql_tzinfo_to_sql --leap tz_file | mysql -u root mysql
当将时区数据装载进mysql中后，可以查看mysql中的时区信息表。

	use mysql;
	show tables like '%time_zone%';
	select * from time_zone;
	select * from time_zone_name;
	select * from time_zone_leap_second;
	select * from time_zone_transition;
	select * from time_zone_transition_type;
一共有五个相关的时区信息表。

那么，此时，可以设置全局变量 `time_zone` 为时区表 `time_zone_name` 中任何一个存在的时区的名字。比如设置为中国标准时间的时区

	mysql> set global time_zone = "Asia/Shanghai";
这种方法设置后，如果 mysql 服务器重启，将不再生效。修改配置文件的方法，才能永久生效。

验证 mysql 的时间

	mysql> select sysdate();
	mysql> select curdate();
	mysql> select now();

__注意：__上述方法修改时区后，退出客户端，重新连接mysql服务器，然后 `show variables like '%time_zone%';` 就可以查看到已经生效。

# 我的问题
楼主是在获取从mysql中获取时间的时候，直接使用 mysql 的函数 `UNIX_TIMESTAMP(VALID_DATE)` 直接将 `VALID_DATE` 时间转化为秒数，该函数转化的为 GMT 的秒数时间，而当我将秒数重新转化成时间字符串的时候，我当前系统的时区设置的为 CST，这个 CST 指的是中国标准时间 (UTC +08:00)，所以前后转换出的时间字符串不一致，相差 8 个小时。

因为之前对时区转换不是很了解，导致发生了这么一个错误，MARK 一下，以后不会再犯这种错误。

参考文章： <br>
1. [time in China](https://en.wikipedia.org/wiki/Time_in_China) <br>
2. [time zone](https://en.wikipedia.org/wiki/Time_in_China) <br>
3. [24时区，GMT，UTC，DST，CST时间详解](https://www.douban.com/note/147740972/) <br>
4. man page