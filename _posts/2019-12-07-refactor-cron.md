---
layout:     post
title:      "代码重构"
subtitle:   "记重写一位初级java写的代码"
date:       2019-12-07
author:     "zark"
header-img: "img/in-post/2016.03/07/post-markdown-introduce.jpg"
tags:
    - 代码重构
---

记重写一位初级java写的代码。看他代码，看的我头痛，一天都还没有完全看懂。最后只能梳理逻辑重写代码。程序员还是要对自己代码有点要求的

大概逻辑：  
- 页面可以配置一个任务的执行计划。可以定制：  
每月工作日、自然日的第n天，或者倒数第n天执行（超过当月天数，以最后一天计算）。并且可以配置是否间隔m小时n分钟执行一次。  
每日几点开始执行并且可以配置是否间隔m小时n分钟执行一次。  
两者都可以配置结束时间（如不配置，默认不结束）


优化点：  
- 添加枚举、引入ScheduleRule模型，方便阅读  
- 添加关键注释（相对于这种纯计算的逻辑，注释体现的很重要）  
- 校验逻辑抽取出来，不与核心逻辑掺杂  
- 避免是用魔法值，去除大量的if-else嵌套，可以提前return来实现  

原代码：
```java
@Component
public class CronUtil {

    public static final String DIMOND_CONFIG_KEY = "rpa_dn_holiday_weekday";

    public static final String HOLIDAY_STRING = "holiday";
    public static final String SWITCHDAY_STRING = "switchday";

    public CronUtil() {
    }

    private List<String> getDays(Object object) {
        if (object == null) {
            return null;
        }
        List<String> list = JSONArray.parseArray(object.toString(), String.class);
        return list;
    }

    private Boolean isWorkday(Calendar calendar) {
        String day = DateUtil.dateToString(calendar.getTime(), DateStyle.YYYYMMDD);
        if (((Set)DiamondUtil.getMapSetConfig(DIMOND_CONFIG_KEY, HOLIDAY_STRING)).contains(day)) {
            return false;
        } else if (((Set)DiamondUtil.getMapSetConfig(DIMOND_CONFIG_KEY, SWITCHDAY_STRING)).contains(day)) {
            return true;
        } else {
            int week = calendar.get(Calendar.DAY_OF_WEEK);
            if (week >= 2 && week <= 6) {
                return true;
            }
        }
        return false;
    }

    private String generateWorkday(String dayRule, Calendar calendar){
        List<String> workdays = new ArrayList<>();
        int monthMaxDay = calendar.getActualMaximum(Calendar.DATE);
        String workday;
        if (dayRule.contains("-") && dayRule.contains("D")) {
            int off = Integer.parseInt(dayRule.substring(dayRule.lastIndexOf("-") + 1));
            for (int i = monthMaxDay; i > 0; i--) {
                calendar.set(Calendar.DATE, i);
                if (isWorkday(calendar)) {
                    String day = DateUtil.dateToString(calendar.getTime(), DateStyle.YYYYMMDD);
                    workdays.add(day);
                }
                if (workdays.size() == off) {
                    break;
                }
            }
            if (workdays.size() < off) {
                workday = workdays.get(workdays.size() - 1);
            } else {
                workday = workdays.get(off - 1);
            }
        } else if (dayRule.contains("D") && !dayRule.contains("-")){
            int off = Integer.parseInt(dayRule.substring(dayRule.lastIndexOf("D") + 1));
            for (int i = 1; i < monthMaxDay + 1; i++) {
                calendar.set(Calendar.DATE, i);
                if (isWorkday(calendar)) {
                    String day = DateUtil.dateToString(calendar.getTime(), DateStyle.YYYYMMDD);
                    workdays.add(day);
                }
                if (workdays.size() == off) {
                    break;
                }
            }
            if (workdays.size() < off) {
                workday = workdays.get(workdays.size() - 1);
            } else {
                workday = workdays.get(off - 1);
            }
        } else if (!dayRule.contains("D") && dayRule.contains("-")){
            String[] strArr = dayRule.replace("-","").split(" ");
            int off = Integer.parseInt(strArr[strArr.length - 1]);
            int maxDay = calendar.getActualMaximum(Calendar.DAY_OF_MONTH);
            if (maxDay - off < 0) {
                off = maxDay;
            }
            calendar.set(Calendar.DAY_OF_MONTH, maxDay - off + 1);
            String day = new SimpleDateFormat("yyyyMMdd").format(calendar.getTime());
            workday = day;
        } else {
            String[] strArr = dayRule.split(" ");
            int off = Integer.parseInt(strArr[strArr.length - 1]);
            if (off > calendar.getActualMaximum(Calendar.DAY_OF_MONTH)) {
                off = calendar.getActualMaximum(Calendar.DAY_OF_MONTH);
            }
            calendar.set(Calendar.DAY_OF_MONTH, off);
            String day = DateUtil.dateToString(calendar.getTime(), DateStyle.YYYYMMDD);
            workday = day;
        }
        return workday;
    }

    private String generateCron(String rule,Calendar calendar) {
        String dayRule = rule.substring(rule.lastIndexOf("D"));
        String workday = generateWorkday(dayRule, calendar);
        String cron = rule.substring(0, rule.lastIndexOf("D")) + workday.substring(6) + " " + workday.substring(4,6) + " ?";
        return cron;
    }
    private Date generateNextExecuteTime(String cron, Date date) {
        CronSequenceGenerator generator = new CronSequenceGenerator(cron);
        date = generator.next(date);
        return date;
    }
    private Date getNextExecuteTimeCron(String triggerRule, Date date) {
        if (triggerRule.contains("D")) {
            Calendar calendar = Calendar.getInstance();
            calendar.setTime(date);
            String cron = generateCron(triggerRule,calendar);
            date = generateNextExecuteTime(cron, date);
            while (date.getTime() - System.currentTimeMillis() > (long) 40 * 24 * 3600 * 1000 ||
                    date.getTime() - System.currentTimeMillis() < 0) {
                if (calendar.get(Calendar.YEAR) != Calendar.getInstance().get(Calendar.YEAR)) {
                    calendar.set(Calendar.YEAR, Calendar.getInstance().get(Calendar.YEAR));
                }
                calendar.add(Calendar.MONTH, 1);
                cron = generateCron(triggerRule, calendar);
                date = generateNextExecuteTime(cron, new Date());
            }
        } else {
            date = generateNextExecuteTime(triggerRule, date);
        }
        return date;
    }

    private String generateAtDay(String dayType, int day, boolean negative) {
        String atDay = "";
        if (dayType.equals("work_day")) {
            atDay = atDay + "D";
        }
        if (negative) {
            atDay = atDay + "-";
        }
        atDay = atDay + day;
        return atDay;
    }

    private Boolean checkOverTime(Date date, String time) {
        if (StringUtils.isEmpty(time)) {
            return false;
        }
        Calendar calendar = generateCalendar(date);
        calendar = setCalendarHourMinuteSecond(calendar, time);
        if (date.getTime() > calendar.getTime().getTime()) {
            return true;
        } else {
            return false;
        }
    }

    private Integer checkDay(Calendar calendar, Calendar otherCalendar) {
        calendar.set(Calendar.HOUR_OF_DAY, 0);
        calendar.set(Calendar.MINUTE, 0);
        calendar.set(Calendar.SECOND, 0);
        calendar.set(Calendar.MILLISECOND, 0);
        otherCalendar.set(Calendar.HOUR_OF_DAY, 0);
        otherCalendar.set(Calendar.MINUTE, 0);
        otherCalendar.set(Calendar.SECOND, 0);
        otherCalendar.set(Calendar.MILLISECOND, 0);
        if (calendar.getTime().getTime() > otherCalendar.getTime().getTime()) {
            return 1;
        } else if (calendar.getTime().getTime() < otherCalendar.getTime().getTime()) {
            return -1;
        } else {
            return 0;
        }
    }

    private Calendar setCalendarHourMinuteSecond(Calendar calendar, String time) {
        calendar.set(Calendar.HOUR_OF_DAY, Integer.parseInt(time.substring(0,2)));
        if (time.length() == 5) {
            calendar.set(Calendar.MINUTE, Integer.parseInt(time.substring(3)));
            calendar.set(Calendar.SECOND, 0);
        }
        if (time.length() == 8) {
            calendar.set(Calendar.MINUTE, Integer.parseInt(time.substring(3,5)));
            calendar.set(Calendar.SECOND, Integer.parseInt(time.substring(7)));
        }
        calendar.set(Calendar.MILLISECOND, 0);
        return calendar;
    }

    private Date setCalendarYearMonthDay(Calendar calendar, String day) {
        calendar.set(Calendar.YEAR, Integer.parseInt(day.substring(0,4)));
        calendar.set(Calendar.MONTH, Integer.parseInt(day.substring(4,6)) - 1);
        calendar.set(Calendar.DAY_OF_MONTH, Integer.parseInt(day.substring(6)));
        return calendar.getTime();
    }

    private Calendar generateCalendar(Date date) {
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(date);
        return calendar;
    }

    private String generateRuleWorkday(Map<String,Object> rule, Calendar calendar) {
        String atDay = generateAtDay((String) rule.get("dayType"), (Integer) rule.get("day"), (Boolean) rule.get("negative"));
        String day = generateWorkday(atDay, calendar);
        return day;
    }

    private Date setNextDayStartDate(Map<String,Object> rule, Date date, Date startDate) {
        String ruleType = (String) rule.get("ruleType");
        Calendar calendarNext = setCalendarHourMinuteSecond(generateCalendar(date), (String) rule.get("startTime"));
        if (ruleType.equals("day")) {
            calendarNext.add(Calendar.DATE, 1);
            date = calendarNext.getTime();
        }
        if (ruleType.equals("month")) {
            calendarNext.add(Calendar.MONTH, 1);
            String day = generateRuleWorkday(rule, calendarNext);
            date = setCalendarYearMonthDay(calendarNext, day);
        }
        return date;
    }

    private Date generateRuleDate(Map<String,Object> rule, Date date, Date startDate) {
        String ruleType = (String) rule.get("ruleType");
        Date dateBak;
        setCalendarHourMinuteSecond(Calendar.getInstance(), DateUtil.dateToString(date, DateStyle.YYYY_MM_DD_HH_MM_SS).substring(11, 16)).getTime();
        if (checkDay(generateCalendar(date), Calendar.getInstance()) <= 0 ) {
            dateBak = setCalendarHourMinuteSecond(Calendar.getInstance(), DateUtil.dateToString(date, DateStyle.YYYY_MM_DD_HH_MM_SS).substring(11, 16)).getTime();
            date = setCalendarHourMinuteSecond(Calendar.getInstance(), (String) rule.get("startTime")).getTime();
        } else {
            dateBak = date;
        }
        String dateToDay = DateUtil.dateToString(date, DateStyle.YYYYMMDD);
        String day = dateToDay;
        if (ruleType.equals("month")) {
            day = generateRuleWorkday(rule, generateCalendar(date));
        }
        if (Integer.parseInt(dateToDay.substring(6)) == Integer.parseInt(day.substring(6))) {
            if (dateBak.getTime() < date.getTime() && date.getTime() > System.currentTimeMillis()) {
                return date;
            }
            Calendar calendarNext = generateCalendar(date);
            while (true) {
                calendarNext.add(Calendar.HOUR_OF_DAY, (Integer) rule.getOrDefault("intervalHours", 0));
                calendarNext.add(Calendar.MINUTE, (Integer) rule.getOrDefault("intervalMinutes", 0));
                if (calendarNext.getTime().getTime() > System.currentTimeMillis() && calendarNext.getTime().getTime() > dateBak.getTime()) {
                    break;
                }
            }
            date = calendarNext.getTime();
            if (calendarNext.get(Calendar.DATE) != generateCalendar(dateBak).get(Calendar.DATE)) {
                date = setNextDayStartDate(rule, dateBak, startDate);
            } else {
                if (checkOverTime(calendarNext.getTime(), (String) rule.get("endTime"))) {
                    date = setNextDayStartDate(rule, dateBak, startDate);
                }
            }
        } else if (Integer.parseInt(dateToDay.substring(6)) > Integer.parseInt(day.substring(6))) {
            date = setNextDayStartDate(rule, date, startDate);
        } else {
            Calendar calendarNext = setCalendarHourMinuteSecond(generateCalendar(date), (String) rule.get("startTime"));
            date = setCalendarYearMonthDay(calendarNext, day);
        }
        if (StringUtils.isEmpty(rule.get("endDate"))) {
            return date;
        } else {
            Date endDate;
            if (StringUtils.isEmpty(rule.get("endTime"))) {
                String endDateTime = rule.get("endDate") + " 23:59:59";
                endDate = DateUtil.stringToDate(endDateTime, DateStyle.YYYY_MM_DD_HH_MM_SS);
            } else {
                String endDateTime = rule.get("endDate") + " " + rule.get("endTime");
                endDate = DateUtil.stringToDate(endDateTime, DateStyle.YYYY_MM_DD_HH_MM);
            }
            if (date.getTime() > endDate.getTime()) {
                return null;
            } else {
                return date;
            }
        }
    }

    private Date getNextExecuteTimeRule(Map<String,Object> rule, Date date) {
        if (date == null || rule == null) {
            return null;
        }
        boolean enable = (Boolean) rule.get("enable");
        if (!enable) {
            return null;
        }
        if (!StringUtils.isEmpty(rule.get("startDate"))) {
            if (((String)rule.get("startDate")).length() != 10
                    || ((String)rule.get("startDate")).split("-").length != 3) {
                return null;
            }
        }
        if (!StringUtils.isEmpty(rule.get("endDate"))) {
            if (((String)rule.get("endDate")).length() != 10
                    || ((String)rule.get("endDate")).split("-").length != 3) {
                return null;
            }
        }
        if (!StringUtils.isEmpty(rule.get("startTime"))) {
            if (((String)rule.get("startTime")).length() != 5
                    || ((String)rule.get("startTime")).split(":").length != 2) {
                return null;
            }
        }
        if (!StringUtils.isEmpty(rule.get("endTime"))) {
            if (((String)rule.get("endTime")).length() != 5
                    || ((String)rule.get("endTime")).split(":").length != 2) {
                return null;
            }
        }
        if (StringUtils.isEmpty(rule.get("startTime"))
                || StringUtils.isEmpty(rule.get("startDate"))) {
            return null;
        }
        boolean interval = (Boolean) rule.get("interval");
        String startDateTime = rule.get("startDate") + " " + rule.get("startTime");
        Date startDate = DateUtil.stringToDate(startDateTime, DateStyle.YYYY_MM_DD_HH_MM);
        if (!StringUtils.isEmpty(rule.get("endDate")) && !StringUtils.isEmpty(rule.get("endTime"))) {
            String endDateTime = rule.get("endDate") + " " + rule.get("endTime");
            Date endDate = DateUtil.stringToDate(endDateTime, DateStyle.YYYY_MM_DD_HH_MM);
            if (startDate.getTime() >= endDate.getTime()) {
                return null;
            }
            if (date.getTime() > endDate.getTime()) {
                return null;
            }
            if (System.currentTimeMillis() > endDate.getTime()) {
                return null;
            }
            Calendar startCalendar = setCalendarHourMinuteSecond(Calendar.getInstance(), (String) rule.get("startTime"));
            Calendar endCalendar = setCalendarHourMinuteSecond(Calendar.getInstance(), (String) rule.get("endTime"));
            if (startCalendar.getTime().getTime() >= endCalendar.getTime().getTime()) {
                return null;
            }
        }
        if((startDate.getTime() > System.currentTimeMillis()) && (startDate.getTime() > date.getTime())) {
            String ruleType = (String) rule.get("ruleType");
            if (ruleType.equals("day")) {
                return startDate;
            }
            if (ruleType.equals("month")) {
                String dateToDay = DateUtil.dateToString(startDate, DateStyle.YYYYMMDD);
                String day = generateRuleWorkday(rule, generateCalendar(startDate));
                if (Integer.parseInt(dateToDay.substring(6)) == Integer.parseInt(day.substring(6))) {
                    return startDate;
                } else {
                    date = startDate;
                }
            }
        }
        if (interval) {
            Integer intervalHours = (Integer) rule.getOrDefault("intervalHours", 0);
            Integer intervalMinutes = (Integer) rule.getOrDefault("intervalMinutes", 0);
            if (intervalHours == 0 && intervalMinutes == 0) {
                date = null;
            } else {
                date = generateRuleDate(rule, date, startDate);
            }
        } else {
            date = null;
        }
        return date;
    }

    public Date getNextExecuteTime(Byte type, String triggerRule, Date date) {
        try {
            if (type.byteValue() == (byte) 1) {
                Map<String,Object> rule = JSONObject.parseObject(triggerRule);
                date = getNextExecuteTimeRule(rule, date);
            } else {
                date = getNextExecuteTimeCron(triggerRule, date);
            }
            return date;
        } catch (Exception e) {
            return null;
        }
    }
}
```

优化后代码：
```java
/**
 * @author wb-zc189961
 * @date 2019-11-25
 */
@Component
public class GetNextExecuteTimeUtil {

    //TODO 之后动态获取
    private Map<String, List<String>> getHolidaySwitchDay() {
        Map<String, List<String>> result = new HashMap<>(2);
        
        //节假日
        result.put("holiday", Lists.newArrayList("20181230", "20181231", "20190101", "20190204", "20190205", "20190206", "20190207", "20190208", "20190209", "20190210",
                "20190405", "20190406", "20190407", "20190501", "20190502", "20190503", "20190504", "20190607", "20190608", "20190609", "20190913",
                "20190914", "20190915", "20191001", "20191002", "20191003", "20191004", "20191005", "20191006", "20191007"));

        //调班 周六周日需要上班
        result.put("switchday", Lists.newArrayList("20181229", "20190202", "20190203", "20190428", "20190505", "20190929", "20191012"));

        return result;
    }


    private Date getCronNextExecTime(ScheduleRule scheduleRule, Date baseTime) {
        //暂时没有这个业务需求
        throw new RpaClientException(RpaCode.SERVER_ERROR, "not support.");
    }


    private Date getExpressionNextExecTime(ScheduleRule scheduleRule, Date baseTime) {
        if (ScheduleConstants.RuleType.DAY.getRuleType().equals(scheduleRule.getRuleType())) {
            return getDayRuleTime(scheduleRule, baseTime);
        } else if (ScheduleConstants.RuleType.MONTH.getRuleType().equals(scheduleRule.getRuleType())) {
            return getMonthRuleTime(scheduleRule, baseTime);
        } else {
            throw new RpaClientException(RpaCode.SERVER_ERROR, "not support.");
        }
    }


    private Date getDayRuleTime(ScheduleRule scheduleRule, Date baseTime) {
        long current = baseTime.getTime();

        //开始时间不会为空，结束时间有可能为空
        Date startTime = mergeDateAndTime(scheduleRule.getStartDate(), scheduleRule.getStartTime());
        Date endTime = StringUtils.isNotEmpty(scheduleRule.getEndDate()) ? mergeDateAndTime(scheduleRule.getEndDate(), scheduleRule.getEndTime()) : null;

        //当前时间大于结束时间
        if (endTime != null && current > endTime.getTime()) {
            return null;
        }

        //开始时间大于当前时间
        if (startTime.getTime() >= current) {
            return startTime;
        }


        Date nextExecuteTime = mergeDateAndTime(DateUtil.dateToString(new Date(), DateStyle.YYYY_MM_DD), scheduleRule.getStartTime());
        if (nextExecuteTime.getTime() > current) {
            return nextExecuteTime;
        }

        //没有间隔执行，今天执行时间已经过了，直接返回第二天的执行时间
        if (!scheduleRule.isInterval()) {
            Date nextDate = DateUtil.addDay(new Date(), 1);
            return mergeDateAndTime(DateUtil.dateToString(nextDate, DateStyle.YYYY_MM_DD), scheduleRule.getStartTime());
        } else {
            //间隔执行，计算当天下次执行时间
            long nextExecuteTimes = nextExecuteTime.getTime();
            long intervalMills = (scheduleRule.getIntervalHours() * 3600 + scheduleRule.getIntervalMinutes() * 60) * 1000;

            while (nextExecuteTimes < current) {
                nextExecuteTimes += intervalMills;
            }

            //有可能间隔时间长，得到的下次执行时间已经超过结束时间
            if (endTime != null && nextExecuteTimes > endTime.getTime()) {
                return null;
            }

            //间隔时间长，直接到了第二天。 直接返回第二天开始时间
            if (!DateUtil.dateToString(new Date(nextExecuteTimes), DateStyle.YYYY_MM_DD).equals(DateUtil.dateToString(baseTime, DateStyle.YYYY_MM_DD))) {
                Date nextDate = DateUtil.addDay(new Date(), 1);
                return mergeDateAndTime(DateUtil.dateToString(nextDate, DateStyle.YYYY_MM_DD), scheduleRule.getStartTime());
            }

            return new Date(nextExecuteTimes);
        }

    }

    private Date mergeDateAndTime(String date, String time) {
        String startDateTime = date + " " + time;
        return DateUtil.stringToDate(startDateTime, DateStyle.YYYY_MM_DD_HH_MM);
    }


    private Date getMonthRuleTime(ScheduleRule scheduleRule, Date baseTime) {
        long current = baseTime.getTime();

        Date effectDate = getEffectDate(scheduleRule, Calendar.getInstance());
        //开始时间不会为空，结束时间有可能为空
        Date startTime = mergeDateAndTime(scheduleRule.getStartDate(), scheduleRule.getStartTime());
        Date endTime = StringUtils.isNotEmpty(scheduleRule.getEndDate()) ? mergeDateAndTime(scheduleRule.getEndDate(), scheduleRule.getEndTime()) : null;

        //当前时间大于结束时间 || 生效时间大于结束时间
        if ((endTime != null && current > endTime.getTime()) || effectDate.getTime() > endTime.getTime()) {
            return null;
        }

        //生效时间大于开始时间 大于当前时间
        while (true) {
            if (effectDate.getTime() >= startTime.getTime() && effectDate.getTime() >= current) {
                return effectDate;
            }
            //生效时间大于开始时间  小于当前时间 & 间隔执行
            if (effectDate.getTime() >= startTime.getTime() && effectDate.getTime() < current && scheduleRule.isInterval()) {
                long intervalMills = (scheduleRule.getIntervalHours() * 3600 + scheduleRule.getIntervalMinutes() * 60) * 1000;
                long nextExecuteTimes = effectDate.getTime();

                while (nextExecuteTimes < current) {
                    nextExecuteTimes += intervalMills;
                }
                if (nextExecuteTimes > endTime.getTime()) {
                    return null;
                }
                //间隔时间很长，直接到了第二天了。直接取下个月的执行时间
                if (!DateUtil.dateToString(new Date(nextExecuteTimes), DateStyle.YYYY_MM_DD).equals(DateUtil.dateToString(effectDate, DateStyle.YYYY_MM_DD))) {
                    effectDate = getNextMonthEffectDate(effectDate, scheduleRule);
                    if (effectDate == null) {
                        return null;
                    }
                }
                return new Date(nextExecuteTimes);
            }
            effectDate = getNextMonthEffectDate(effectDate, scheduleRule);
            if (effectDate == null) {
                return null;
            }
        }
    }

    private Date getNextMonthEffectDate(Date currMonthEffectDate, ScheduleRule scheduleRule) {
        Date nextMonthDate = DateUtil.addMonth(currMonthEffectDate, 1);
        Calendar calendar = Calendar.getInstance();
        calendar.setTime(nextMonthDate);

        Date nextMonthEffectDate = getEffectDate(scheduleRule, calendar);

        Date endTime = StringUtils.isNotEmpty(scheduleRule.getEndDate()) ? mergeDateAndTime(scheduleRule.getEndDate(), scheduleRule.getEndTime()) : null;

        if (endTime != null && nextMonthEffectDate.getTime() > endTime.getTime()) {
            return null;
        }
        return nextMonthEffectDate;
    }

    private Date getEffectDate(ScheduleRule scheduleRule, Calendar calendar) {
        Date effectDate;

        if (ScheduleConstants.DateType.DAY.getDateType().equals(scheduleRule.getDayType())) {
            //自然日
            effectDate = getNatureEffectDate(scheduleRule, calendar);
        } else if (ScheduleConstants.DateType.WORK_DAY.getDateType().equals(scheduleRule.getDayType())) {
            //工作日
            effectDate = getWorkdayEffectDate(scheduleRule, calendar);
        } else {
            throw new RpaClientException(RpaCode.SERVER_ERROR, "not support.");
        }
        return effectDate;
    }


    /**
     * 计算倒数第n个自然日、顺数第n个自然日
     */
    private Date getNatureEffectDate(ScheduleRule scheduleRule, Calendar calendar) {
        int maxDay = calendar.getActualMaximum(Calendar.DATE);
        //超过月份最大日期，取最后一日
        if (maxDay < Integer.parseInt(scheduleRule.getDay())) {
            scheduleRule.setDay(String.valueOf(maxDay));
        }
        if (scheduleRule.isNegative()) {
            calendar.set(Calendar.DAY_OF_MONTH, (maxDay - Integer.parseInt(scheduleRule.getDay())) + 1);
        } else {
            calendar.set(Calendar.DAY_OF_MONTH, Integer.parseInt(scheduleRule.getDay()));
        }
        buildTime(calendar, scheduleRule);
        return calendar.getTime();
    }

    /**
     * 计算倒数第n个工作日、顺数第n个工作日
     */
    private Date getWorkdayEffectDate(ScheduleRule scheduleRule, Calendar calendar) {
        int i = 0;
        int maxDay = calendar.getActualMaximum(Calendar.DATE);

        if (scheduleRule.isNegative()) {
            int firstWorkday = 30;
            for (int j = maxDay; j > 0; j--) {
                calendar.set(Calendar.DAY_OF_MONTH, j);
                i = isWorkday(calendar) ? i + 1 : i;

                if (isWorkday(calendar)) {
                    firstWorkday = j;
                }

                if (i == Integer.parseInt(scheduleRule.getDay())) {
                    break;
                }
            }

            if (i < Integer.parseInt(scheduleRule.getDay())) {
                //取第一个工作日
                calendar.set(Calendar.DAY_OF_MONTH, firstWorkday);
            }
        } else {
            int lastWorkday = 1;

            for (int j = 0; j < maxDay; j++) {
                calendar.set(Calendar.DAY_OF_MONTH, j + 1);
                i = isWorkday(calendar) ? i + 1 : i;

                if (isWorkday(calendar)) {
                    lastWorkday = j + 1;
                }

                if (i == Integer.parseInt(scheduleRule.getDay())) {
                    break;
                }
            }
            if (i < Integer.parseInt(scheduleRule.getDay())) {
                //取最后一个工作日
                calendar.set(Calendar.DAY_OF_MONTH, lastWorkday);
            }
        }

        buildTime(calendar, scheduleRule);
        return calendar.getTime();
    }

    private void buildTime(Calendar calendar, ScheduleRule scheduleRule) {
        String startTime = scheduleRule.getStartTime();

        String[] hourMinus = startTime.split(":");

        calendar.set(Calendar.HOUR_OF_DAY, Integer.parseInt(hourMinus[0]));
        calendar.set(Calendar.MINUTE, Integer.parseInt(hourMinus[1]));
        calendar.set(Calendar.SECOND, 0);
    }


    private boolean isWorkday(Calendar calendar) {
        String day = DateUtil.dateToString(calendar.getTime(), DateStyle.YYYYMMDD);
        Map<String, List<String>> holidaySwitchDay = getHolidaySwitchDay();

        if (holidaySwitchDay.get("holiday").contains(day)) {
            return false;
        } else if (holidaySwitchDay.get("switchday").contains(day)) {
            return true;
        } else {
            int week = calendar.get(Calendar.DAY_OF_WEEK);
            if (week >= 2 && week <= 6) {
                return true;
            }
        }
        return false;
    }

    //scheduleRule的参数校验逻辑，统统放在入口处校验，这里不校验
    //baseTime:基准线时间。默认是当前时间，便于根据nextExecuteTime计算 下次的nextExecuteTime(第二次执行时间)
    public Date getNextExecuteTime(ScheduleConstants.Type type, ScheduleRule scheduleRule, Date baseTime) {
        baseTime = baseTime == null ? new Date() : baseTime;
        if (ScheduleConstants.Type.CRON_EXPRESSION.equals(type)) {
            return getCronNextExecTime(scheduleRule, baseTime);
        } else if (ScheduleConstants.Type.CUSTOM_EXPRESSION.equals(type)) {
            return getExpressionNextExecTime(scheduleRule, baseTime);
        } else {
            throw new RpaClientException(RpaCode.SERVER_ERROR, "not support.");
        }
    }
}
```
附pojo：
ScheduleConstants:
```java
import lombok.Getter;

/**
 * @author wb-zc189961
 * @date 2019-11-22
 */
public interface ScheduleConstants {

    enum Type {
        /**
         * cron表达式
         */
        CRON_EXPRESSION((byte) 0),

        /**
         * 用户页面自定义设置
         */
        CUSTOM_EXPRESSION((byte) 1);

        @Getter
        private byte type;

        Type(byte type) {
            this.type = type;
        }

        public static Type getType(byte b) {
            Type[] values = Type.values();
            for (Type value : values) {
                if (value.getType() == b) {
                    return value;
                }
            }
            return null;
        }
    }

    enum RuleType {
        /**
         * 日计划
         */
        DAY("day"),

        /**
         * 月计划
         */
        MONTH("month");

        @Getter
        private String ruleType;

        RuleType(String ruleType) {
            this.ruleType = ruleType;
        }

        public static RuleType getRuleType(String ruleType) {
            RuleType[] values = RuleType.values();
            for (RuleType value : values) {
                if (value.getRuleType().equals(ruleType)) {
                    return value;
                }
            }
            return null;
        }
    }

    enum DateType {
        /**
         * 工作日
         */
        WORK_DAY("work_day"),

        /**
         * 自然日
         */
        DAY("day");

        @Getter
        private String dateType;

        DateType(String dateType) {
            this.dateType = dateType;
        }
    }
}
```

ScheduleRule:
```java
import lombok.Data;
import lombok.ToString;

/**
 * @author wb-zc189961
 * @date 2019-11-22
 */
@Data
@ToString
public class ScheduleRule {

    /**
     * 是否启用
     */
    private boolean enable;

    /**
     * 开始日期
     */
    private String startDate;

    /**
     * 开始时间
     */
    private String startTime;

    /**
     * 结束日期
     */
    private String endDate;

    /**
     * 结束时间
     */
    private String endTime;

    /**
     * 是否间隔执行
     */
    private boolean interval;

    /**
     * 间隔多少小时
     */
    private Integer intervalHours;

    /**
     * 间隔多少分钟
     */
    private Integer intervalMinutes;


    /**
     * 计划类型：月计划 、 日计划
     *
     * @see com.alibaba.ursa.rpa.orchestrator.model.enums.ScheduleConstants.RuleType
     */
    private String ruleType;

    /**
     * 月计划，每月哪天执行（月计划才有属性）
     */
    private String day;

    /**
     * 日期类型：自然日，工作日（月计划才有属性）
     *
     * @see com.alibaba.ursa.rpa.orchestrator.model.enums.ScheduleConstants.DateType
     */
    private String dayType;

    /**
     * 是否倒数
     */
    private boolean negative;
}
```