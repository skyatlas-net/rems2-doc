# REMS2 触发器表达式中函数

|Function	|Desc	|Parameters	|Comments|
|-----------|-------|-----------|--------|
|abschange	|最后两次值的变化绝对值|		|"支持类型：float，int，str，text，log 如果是字符：0 - 值相等； 1 - 值不相等"|
|avg(sec\|#num,<time_shift>)|	指定时间段内，监控项的平均值|	"sec 或者 #num，指定评估时间段（sec是秒，#num是最近值个数）time_shift（可选）评估时间段向前移动的秒"|	"支持类型： float，int示例：avg(#5) 最后5个值的平均 avg(1h) 最近一个小时的平均 avg(1h, 1d) 一天之前的一小时平均		"|