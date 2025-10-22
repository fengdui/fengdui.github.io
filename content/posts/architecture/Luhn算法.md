---
title: "liquibase管理数据库版本"
date: "2025-10-22"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


![img_19.png](/pic/img_19.png)

serial number自测即可 其他都定死 最后一位用下面的算法计算 得到应该序列号

```
/**
     * 计算 Luhn 校验位
     *
     * @param number 不含校验位的数字字符串
     * @return 校验位 (0-9)
     */
    public static int calculateLuhnCheckDigit(String number) {
        int sum = 0;
        boolean alternate = true;

        // 从右到左处理每一位
        for (int i = number.length() - 1; i >= 0; i--) {
            int digit = Character.getNumericValue(number.charAt(i));

            if (alternate) {
                digit *= 2;
                if (digit > 9) {
                    digit = digit / 10 + digit % 10;
                }
            }
            sum += digit;
            alternate = !alternate;
        }

        return (10 - (sum % 10)) % 10;
    }

    /**
     * 生成带 Luhn 校验位的完整号码
     *
     * @param number 不含校验位的数字字符串
     * @return 带校验位的完整号码
     */
    public static String generateLuhnNumber(String number) {
        int checkDigit = calculateLuhnCheckDigit(number);
        return number + checkDigit;
    }

    /**
     * 匹配银行卡
     *
     * @param cardNo
     * @return
     */
    public static boolean matchLuhn(String cardNo) {
        try {
            int[] cardNoArr = new int[cardNo.length()];
            for (int i = 0; i < cardNo.length(); i++) {
                cardNoArr[i] = Integer.valueOf(String.valueOf(cardNo.charAt(i)));
            }
            for (int i = cardNoArr.length - 2; i >= 0; i -= 2) {
                cardNoArr[i] <<= 1;
                cardNoArr[i] = cardNoArr[i] / 10 + cardNoArr[i] % 10;
            }
            int sum = 0;
            for (int i = 0; i < cardNoArr.length; i++) {
                sum += cardNoArr[i];
            }
            return sum % 10 == 0;
        } catch (Exception e) {
            return false;
        }
    }
```
