---
title: django时区问题
date: 2022-09-04 06:15:31
tags:
  - django
---
问题:
启用了时区, 在原生`sql`中没有转换时区, 导致时间筛选不正确.

原因:
是数据库保存的是`UTC`的时间.

解决办法:
1. 没有多时区用户, 直接不启用时区
2. 启用时区, `orm`,表单会自动处理时区, 原生`sql`在执行前自己把时间转换为`UTC`时间
3. 启用时区, 数据库连接配置`DATABASES`里面设置`TIME_ZONE`
4. 启用时区, 原生`sql`里面使用`AT TIME ZONE` 

因为已经有数据了, 而且不同数据库转换时间的方法不一致, 为了通用所以目前使用的是方法2
```python
from django.utils import timezone
from django.utils.dateparse import parse_datetime

def utc_time(datetime_str):
    naive = parse_datetime(datetime_str)
    current_tz = timezone.get_current_timezone()
    ut = naive.replace(tzinfo=current_tz).astimezone(timezone.utc)
    return ut.strftime("%Y-%m-%d %H:%M:%S")
```

## 时区
当启用对时区的支持时，Django 在数据库中以 UTC 为单位存储日期时间信息，在内部使用具有时区的日期时间对象，并在模板和表单中将其转换为最终用户的时区。

`aware` 代表带有时区, `naive` 代表没有带时区.

## 表单的处理
在表单`Field`里面的`clean`方法把提交时间转换为带时区的时间, 同理渲染的时候还有`to_current_timezone`, 把数据库utc时间转换为当前时区时间.
```python
# django/forms/utils.py
def from_current_timezone(value):
    """
    When time zone support is enabled, convert naive datetimes
    entered in the current time zone to aware datetimes.
    """
    if settings.USE_TZ and value is not None and timezone.is_naive(value):
        current_timezone = timezone.get_current_timezone()
        try:
            if not timezone._is_pytz_zone(
                current_timezone
            ) and timezone._datetime_ambiguous_or_imaginary(value, current_timezone):
                raise ValueError("Ambiguous or non-existent time.")
            return timezone.make_aware(value, current_timezone)
        except Exception as exc:
            raise ValidationError(
                _(
                    "%(datetime)s couldn’t be interpreted "
                    "in time zone %(current_timezone)s; it "
                    "may be ambiguous or it may not exist."
                ),
                code="ambiguous_timezone",
                params={"datetime": value, "current_timezone": current_timezone},
            ) from exc
    return value
```

## orm
orm 执行前也会把带有当前时区的转换为数据的时区. 每种数据库都实现了该方法.
```python
# django/db/models/function/datetime.py
class Extract(TimezoneMixin, Transform):
    ...
    def as_sql(self, compiler, connection):
        sql, params = compiler.compile(self.lhs)
        lhs_output_field = self.lhs.output_field
        if isinstance(lhs_output_field, DateTimeField):
            tzname = self.get_tzname()
            sql, params = connection.ops.datetime_extract_sql(
                self.lookup_name, sql, tuple(params), tzname
            )
            
            
            
# django/db/backends/base/operations.py
def datetime_extract_sql(self, lookup_type, sql, params, tzname):
    """
    Given a lookup_type of 'year', 'month', 'day', 'hour', 'minute', or
    'second', return the SQL that extracts a value from the given
    datetime field field_name.
    """
    raise NotImplementedError(
        "subclasses of BaseDatabaseOperations may require a datetime_extract_sql() "
        "method"
    )
```


## 参考链接
- [django官方文档 时区](https://docs.djangoproject.com/zh-hans/4.0/topics/i18n/timezones/)
- [django官方文档 settings time-zone](https://docs.djangoproject.com/zh-hans/4.1/ref/settings/#time-zone)
- [django官方文档 trunc](https://docs.djangoproject.com/zh-hans/4.1/ref/models/database-functions/#trunc)