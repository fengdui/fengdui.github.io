---
title: "guava限流"
date: "2017-12-18"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

简单版本
```
import com.google.common.util.concurrent.RateLimiter;

public class RateLimit {
    private static RateLimiter one = RateLimiter.create(2);//每秒2个
    private static RateLimiter two = RateLimiter.create(2);//每秒2个

    public static void acquire(RateLimiter r, int num){
        double time = r.acquire(num);
        System.out.println("wait time="+time);
    }

    public static void main(String[] args) throws InterruptedException {
        acquire(one,1);
        acquire(one,1);
        acquire(one,1);
        System.out.println("-----");
        acquire(two,10);
        acquire(two,1);

    }
}
```

进阶版本
```
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class GuavaRateLimiter {

    private static final Logger logger = LoggerFactory.getLogger(GuavaRateLimiter.class);

    private volatile List<Limiter> limiters;

    private volatile RateExceedAction rateExceedAction;

    private static final String RATE_LIMIT_KEY = "limit";

    private static final String RATE_EXCEED_KEY = "exceed";

    enum RateExceedAction {warning, disconnect, responseError}

    /**
     * 增加计数器并检查是否到达限制
     *
     * @return true 到达限制
     */
    boolean incrementAndCheck(String id) {
        try {
            boolean exceed = false;
            for (Limiter limiter : limiters) {
                exceed = limiter.incrementAndCheck(id) || exceed;
            }
            return exceed;
        } catch (Throwable t) {
            logger.error("unexpected limiter error, msg:{}", t.getMessage(), t);
            return false;
        }
    }

    @Override
    public String toString() {
        return "RateLimiter{" +
                "limiters=" + limiters +
                ", rateExceedAction=" + rateExceedAction +
                '}';
    }

    private static class Limiter {
        private int durationInSec;

        private int limit;

        private LoadingCache<String, AtomicInteger> bucket;

        Limiter(String setting) {
            if (StringUtils.isBlank(setting) || !setting.matches("^\\d+,\\d+$")) {
                logger.error("invalid setting for limiter: {}, fallback to default", setting);
                durationInSec = 60;
                limit = 200;
            } else {
                String[] split = setting.split(",");
                durationInSec = Integer.valueOf(split[0]);
                limit = Integer.valueOf(split[1]);
            }

            bucket = CacheBuilder.newBuilder().expireAfterWrite(durationInSec * 2, TimeUnit.SECONDS).build(new CacheLoader<String, AtomicInteger>() {
                @Override
                public AtomicInteger load(@SuppressWarnings("NullableProblems") String key) {
                    return new AtomicInteger();
                }
            });
        }

        /**
         * 增加计数器并检查是否到达限制
         */
        boolean incrementAndCheck(String id) {
            String bucketKey = id + "_" + Instant.now().getEpochSecond() / durationInSec;
            if (bucket.getUnchecked(bucketKey).get() > limit) {
                return true;
            } else {
                boolean exceed = bucket.getUnchecked(bucketKey).incrementAndGet() > limit;
                if (exceed) {
                    logger.warn("{} exceed limit:{} in duration:{}", id, limit, durationInSec);
                }
                return exceed;
            }
        }

        @Override
        public String toString() {
            return "Limiter{" +
                    "durationInSec=" + durationInSec +
                    ", limit=" + limit +
                    '}';
        }
    }
}
```