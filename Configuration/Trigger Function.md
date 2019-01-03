# REMS2 触发器表达式中函数

|Function	|Desc	|Parameters	|Comments|
|-----------|-------|-----------|--------|
|abschange	|最后两次值的变化绝对值|		|"支持类型：float，int，str，text，log 如果是字符：0 - 值相等； 1 - 值不相等"|
|avg(sec\|#num,\<time_shift\>)|	指定时间段内，监控项的平均值|	"sec 或者 #num，指定评估时间段（sec是秒，#num是最近值个数）time_shift（可选）评估时间段向前移动的秒"|	"支持类型： float，int示例：avg(#5) 最后5个值的平均 avg(1h) 最近一个小时的平均 avg(1h, 1d) 一天之前的一小时平均		"|
|band(sec\|#num, mask, \<time_shift\>)|	对监控项的值，执行bit and操作|	"sec（可省略），#num表示最近的第num个值 mask 掩码 64位无符号整型 time_shift …"|	"支持值类型： int 注意： 这里的#num和其他函数中不同，这里指最近的第num个	"|
|change	|最后一个值与倒数第二个值的差|		|"支持类型：float，int，str，text，log For example: (previous value;last value=change) 1;5=+4 3;1=-2 0;-2.5=-2.5 如果是字符：0 - 值相等； 1 - 值不相等"|
|count(sec\|#num, \<pattern\>, \<operator\>,\<time_shift\>)|	指定时间段内，监控项值的个数|	"sec 或者 #num，指定评估时间段（sec是秒，#num是最近值个数）pattern 可选 operator 可选：支持的operator：eq - 相等 ne - 不相等 gt - 大于 ge - 大于等于 lt - 小于 le - 小于等于 like - 匹配 pattern (case-sensitive) band - bitwise AND regexp - 大小写敏感的正则表达式匹配 匹配的正则表达式由 pattern提供 iregexp - 大小写不敏感的正则表达式匹配 匹配的正则表达式由 pattern提供 注意：eq (default), ne, gt, ge, lt, le, band, regexp, iregexp支持int型监控项 eq (default), ne, gt, ge, lt, le, regexp, iregexp 支持 float 监控项 like (default), eq, ne, egexp, iregexp 支持 string, text and log 型监控项 time_shift（可选），同avg()|"	"支持的数据类型： float, integer, string, text, log float精度： 0.000001 如果第三个参数operator是band，第二个参数pattern需要是以’/‘ 分隔的两个数字： \[number_to_compare_with\]/\[mask\], 先把监控项值与mask做bit and，其结果与number_to_compare_with对比，如果相等这个监控项值被count函数计数。如果number_to_compare_with和mask相等，则pattern中只需要提供mask值（不需要’/‘) 如果operator参数是regexp或者iregexp，pattern参数是常规正则表达式或者系统中预定义的全局正则表达式(@符号开头) Examples: ⇒ count(10m) → 最后十分钟，监控项值个数 ⇒ count(10m,"error",eq) → 最后十分钟，监控项值等于error的个数 ⇒ count(10m,12) → 最后十分钟，监控项值等于12的个数 ⇒ count(10m,12,gt) → 最后十分钟，监控项值大于12的个数 ⇒ count(#10,12,gt) → 最后十个监控项值中，大于12的个数⇒ count(10m,12,gt,1d) → 一天之前的这10分钟，监控项值大于12的个数 ⇒ count(10m,6/7,band) → 最后十分钟，监控项值最后三个bit位为110的值个数. ⇒ count(10m,,,1d) → 一天前的这十分钟里，监控项值个数"|
|date|	当前日期：YYYYMMDD格式		|||
|dayofmonth|	day of month： 1-31		|||
|dayofweek|	"day of week： 1-7 Mon-1，Sun-7"		|||
|delta(sec\|#num, \<time_shift\>)|	指定时间段内，监控项的最大值 - 最小值|		|支持数据类型： float ，int|
|diff|	检查最后两次返回值是否相同||		"1 - 不同 0 - 相同"|
|forecast(sec\|#num,\<time_shift\>,time,\<fit\>,\<mode\>)	|Future value, max, min, delta or avg of the item	|"sec 或者 #num，指定评估时间段（sec是秒，#num是最近值个数）\n time_shift, 同avg() \n time, forecasting horizon in seconds \nfit, function used to fit historical data \n 支持的fits：linear - 线性函数 \n polynomialN - N 度多项式(1 <= N <= 6) \n exponential - 指数函数 \n logarithmic - 对数函数 \n power - 幂函数 \n 注意：默认是linear线性函数，polynomial1等价于线性函数 \n mode(可选): 输出要求 \n 支持的mode: \n value - 默认 \n max - 最大值 \n min - 最小值 \n delta - max-min \n avg - 平均值 \n value 预测 now + time时刻监控项的值 \n max，min，delta，avg预测now到now + time时间段内"|	"支持数据类型： float， int \n 返回值范围： 999999999999.9999，-999999999999.9999 \n 如果表达式misused - not supported \n 有错的话 返回 -1 \n Examples：\n ⇒ forecast(#10,,1h) → 基于最近10个值，预测监控项1小时后的值 \n ⇒ forecast(1h,,30m) → 基于过去1小时的数据，预测监控项30分钟后的值\n ⇒ forecast(1h,1d,12h) → 基于1天前最近1小时时段的数据，预测12小时后监控项的数值 \n ⇒ forecast(1h,,10m,exponential) → 基于过去1小时数据，使用子数函数预测10分钟后监控项的数值 \n ⇒ forecast(1h,,2h,polynomial3,max) → 基于过去1小时数据，使用3次多项式函数预测未来2小时内，监控项可能达到的最大值 \n ⇒ forecast(# 2,, -20m) → 基于过去两个值，估算20分钟监控项的值"|
|fuzzytime(sec)|	检查监控项的时间戳值与Rems2 Server时间的差|	sec - 秒数	|"支持数据类型： float，int 0 - 如果时间差超过 sec 1 - 否则 常用于system.localtime   vfs.file.time [/path/file, modify]一起使用 Example ⇒ fuzzytime(60)=0 → 检查时间差超过60秒"|
|iregexp(pattern, \<sec\|#num\>)|	大小写不敏感的regexp|		|支持的数据类型：str，log，text|
|last(sec\|#num, \<time_shift\>)|	最近的监控项值|	#num 最近、倒数第N个值|	"支持数据类型：float，int，str，text，log  For example: last() is always equal to last(#1) last(#3) - 倒数第三个返回值 (而不是最后的3个值)"|
|logeventid(pattern)|	检查最后一条log检查项的Event ID是否匹配正则表达式pattern|	pattern： 正则表达式|	"支持数据类型： log 0 - 不匹配 1 - 匹配"|
|logseverity|	最后一条日志检查项的severity|		|"支持的类型： log 0 - 默认severity N - Windows event logs: 1 - Information, 2 - Warning, 4 - Error, 7 - Failure Audit, 8 - Success Audit, 9 - Critical, 10 - Verbose"|
|logsource(pattern)|	检查日志检查项log source是否匹配pattern|		|"0 - 不匹配 1 - 匹配 一般用于Windows 日志，如logsource(‘VMware Server’)"|
|max(sec\|#num, \<time_shift\>)|	指定时间段，监控项返回值的最大值|		|支持数据类型：float，int|
|min(sec\|#num, \<time_shift\>)|	指定时间段，监控项返回值的最小值|		|支持数据类型：float，int|
|nodata(sec)|	检查未收到返回数据|	"sec 秒数 不要小于30秒"|	"1 - 指定时间内，未收到数据0 - 收到"|
|now|	Unix time		|||
|percentile(sec\|#num, \<time_shift\>, percentage)|	一个周期的第P百分位数，其中P（百分比）由第三个参数指定。|		|95th常用，表示指定时段，95%的监控项返回值都不超过的统计值|
|prev|	倒数第二个值|		|last(#2)|
|regexp(pattern, \<sec\|#num\>)|	检查最近的返回数值（可能多个）是否匹配正则表达式——大小写敏感|	"patter - 正则表达式 sec or #num - 最近sec秒或者最近num个返回值"|	"支持的类型： str，text，log 返回值： 1 - 匹配 0 - 不匹配"|
|str(pattern, \<sec\|#num\>)|	检查最近的返回数值（可能多个）是否包含字符——大小写敏感	|"patter - 普通字符 ; sec or #num - 最近sec秒或者最近num个返回值"|	"支持的类型： str，text，log 返回值： 1 - 匹配; 0 - 不匹配"|
|strlen(sec\|#num, \<time_shift\>)|	字符长度|	#num - 倒数 Nth 返回值|	"Examples: ⇒ strlen()(等同 strlen(#1)) →最后一个返回值的字符长度 ⇒ strlen(#3) → 倒数第三个返回值的字符长度 ⇒ strlen(,1d) → 一天之前最后一个返回值的字符长度."|
|sum(sec\|#num, \<time_shift\>)|	指定时间段内，监控项返回数值的和|		|支持类型： float，int|
|time|	返回当前时间 HHMMSS 格式		|||
|timeleft(sec\|#num,\<time_shift\>,threshold,\<fit\>)|	监控项值还有多少秒能触及阀值|	"sec\|#num 指定时间段，函数会基于这个时段数据来预估 time_shift 同avg() threshold 阀值fit （可选） 同forecast函数"|	"支持数据类型： float，int 返回值范围： <=999999999999.9999 unsupported - misused -1 - error Examples: ⇒ timeleft(#10,,0) → 基于最近的10个数据，预测监控项触及阀值0 还需时间 ⇒ timeleft(1h,,100) → 基于过去1小时数据，预估触及阀值100还需时间 ⇒ timeleft(1h,1d,0) → 基于1天前的1小时数据，评估触及阀值0需要的时间 ⇒ timeleft(1h,,200,polynomial2) → time until item reaches 200 based on last hour data and assumption that item behaves like quadratic (second degree) polynomial"|