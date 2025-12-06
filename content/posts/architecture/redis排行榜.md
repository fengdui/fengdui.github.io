---
title: "redis排行榜"
date: "2019-05-21"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
要做一个答题活动 单机qps 2000  
redis sorted set 实现  
添加或更新分数	ZADD key score member	添加成员及分数，若成员存在则更新分数。例：ZADD leaderboard 89 user1  
获取成员分数	ZSCORE key member	查询指定成员的分数。例：ZSCORE leaderboard user1  
查询排名	ZREVRANK key member	查询成员在降序排行榜中的位次（从0开始）。例：ZREVRANK leaderboard user1  
增加分数	ZINCRBY key increment member	为成员的分数增加一个增量（可为负）。例：ZINCRBY leaderboard 5 user1  
获取Top N	ZREVRANGE key start stop [WITHSCORES]	获取降序排名中指定范围内的成员。例：ZREVRANGE leaderboard 0 9 WITHSCORES （查看前十名）  
删除成员	ZREM key member	从排行榜中移除指定成员。  

排名唯一  
最终分数 = 实际分数 * 10^N + (MAX_TIMESTAMP - 时间戳)  
如果排名相同，时间戳大的排名更靠前。  
如果允许同分，无法直接用ZREVRANK。例如，查询成员B（与C同分）的排名时，需要先找到同分中排名最靠前的成员（C），再查询该成员的排名  
