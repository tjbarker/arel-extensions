# Arel Extensions

![GitHub workflow](https://github.com/Faveod/arel-extensions/actions/workflows/ruby.yml/badge.svg)
[![AppVeyor Build Status](https://img.shields.io/appveyor/ci/jdelporte/arel-extensions.svg?label=AppVeyor%20build)](https://ci.appveyor.com/project/jdelporte/arel-extensions)
![](http://img.shields.io/badge/license-MIT-brightgreen.svg)

Gem: [![Latest Release](https://img.shields.io/gem/v/arel_extensions.svg)](https://rubygems.org/gems/arel_extensions)
[![Gem](https://ruby-gem-downloads-badge.herokuapp.com/arel_extensions?type=total)](https://rubygems.org/gems/arel_extensions)
[![Gem](https://ruby-gem-downloads-badge.herokuapp.com/arel_extensions?label=downloads-current-version)](https://rubygems.org/gems/arel_extensions)

Arel Extensions adds shortcuts, fixes and new ORM mappings (Ruby to SQL) to Arel.
It aims to ensure pure Ruby syntax for most usual cases.
It allows to use more advanced SQL functions for any supported RDBMS.


## Requirements

Arel 6 (Rails 4) or Arel 7+ (Rails 5).
[Arel Repository](http://github.com/rails/arel)

or

Rails 6
[Rails Repository](http://github.com/rails/rails)

## Usage

Most of the features will work just by adding the gem to your Gemfiles. To make sure to get all the features for any dbms, you should execute the next line as soon as you get your connection to your DB:

```ruby
ArelExtensions::CommonSqlFunctions.new(ActiveRecord::Base.connection).add_sql_functions()
```

It will add common SQL features in your DB to align ti with current routines. Technically, it will execute SQL scripts from init folder.


## Examples

t is an Arel::Table for table my_table

## Comparators

```ruby
(t[:date1] > t[:date2]).to_sql # (same as (t[:date1].gt(t[:date2])).to_sql)
# => my_table.date1 > my_table.date2
```

```ruby
(t[:nb] > 42).to_sql # (same as (t[:nb].gt(42)).to_sql)
# => my_table.nb > 42
```

Other operators : <, >=, <=, =~


## Maths

Currently in Arel:
```ruby
(t[:nb] + 42).to_sql
# => my_table.nb + 42
```

But:
```ruby
(t[:nb].sum + 42).to_sql
# => NoMethodError: undefined method `+' for #<Arel::Nodes::Sum>
```

With Arel Extensions:
```ruby
(t[:nb].sum + 42).to_sql
# => SUM(my_table.nb) + 42
```

Other functions : ABS, RAND, ROUND, FLOOR, CEIL, FORMAT

For Example:
```ruby
t[:price].format_number("%07.2f €","fr_FR")
# equivalent to 'sprintf("%07.2f €",price)' plus locale management
```

## String operations

```ruby
(t[:name] + ' append').to_sql
# => CONCAT(my_table.name, ' append')

(t[:name].coalesce('default')).to_sql
# => COALESCE(my_table.name, 'default')

(t[:name].blank).to_sql
# => TRIM(TRIM(TRIM(COALESCE(my_table.name, '')), '\t'), '\n') = ''

(t[:name] =~ /\A[a-d_]+/).to_sql
# => my_table.name REGEXP '\^[a-d_]+'
```

Other functions : SOUNDEX, LENGTH, REPLACE, LOCATE, SUBSTRING, TRIM

### String Array operations

```t[:list]``` is a classical varchar containing a comma separated list ("1,2,3,4")

```ruby
(t[:list] & 3).to_sql
# => FIND_IN_SET('3', my_table.list)

(t[:list] & [2,3]).to_sql
# => FIND_IN_SET('2', my_table.list) OR FIND_IN_SET('3', my_table.list)
```


## Date & Time operations

```ruby
(t[:birthdate] + 10.years).to_sql
# => ADDDATE(my_table.birthdate, INTERVAL 10 YEAR)

((t[:birthdate] - Date.today) * -1).to_sql
# => DATEDIFF(my_table.birthdate, '2017-01-01') * -1

t[:birthdate].week.to_sql
# => WEEK(my_table.birthdate)

t[:birthdate].month.to_sql
# => MONTH(my_table.birthdate)

t[:birthdate].year.to_sql
# => YEAR(my_table.birthdate)
```

### Datetime

```ruby
# datetime difference
t[:birthdate] - Time.utc(2014, 3, 3, 12, 41, 18)

# comparison
t[:birthdate] >= '2014-03-03 10:10:10'
```

### Format and Time Zone Conversion

`format` has two forms:

```ruby
t[:birthdate].format('%Y-%m-%d').to_sql
# => DATE_FORMAT(my_table.birthdate, '%Y-%m-%d')
```

Which formats the datetime without any time zone conversion.
The second form accepts 2 kinds of values:

1. String:

```ruby
t[:birthdate].format('%Y/%m/%d %H:%M:%S', 'posix/Pacific/Tahiti')
# => DATE_FORMAT(CONVERT_TZ(CAST(my_table.birthdate AS datetime), 'UTC', 'posix/Pacific/Tahiti'), '%Y/%m/%d %H:%i:%S')          ## MySQL
# => TO_CHAR(CAST(my_table.birthdate AS timestamp with time zone) AT TIME ZONE 'posix/Pacific/Tahiti', 'YYYY/MM/DD HH24:MI:SS') ## PostgreSQL
# => CONVERT(datetime, my_table.birthdate) AT TIME ZONE 'UTC' AT TIME ZONE N'posix/Pacific/Tahiti'                              ## SQL Server (& truncated for clarity)
#                                                                            ^^^^^^^^^^^^^^^^^^^^ 🚨 Invalid timezone for SQL Server. Explanation below.
```

which will convert the datetime field to the supplied time zone. This generally
means that you're letting the RDBMS decide or infer what is the timezone of the
column before conversion to the supplied timezone.

1. Hash of the form `{ src_time_zone => dst_time_zone }`:

```ruby
t[:birthdate].format('%Y/%m/%d %H:%M:%S', { 'posix/Europe/Paris' => 'posix/Pacific/Tahiti' })
```

which will explicitly indicate the original timestamp that should be considered
by the RDBMS.

Warning:

- ⚠️ Time Zone names are specific to each RDBMS. While `PostgreSQL` and `MySQL`
  have overlaping names (the ones prefixed with `posix`), you should always
  read your vendor's documentation. `SQL Server` is a black sheep and has its
  own conventions.
- ⚠️ Daylight saving is managed by the RDBMS vendor. Choose the approptiate time
  zone name that enforces proper daylight saving conversions.
  - ☣️ Choosing `GMT+offset` will certainly bypass daylight saving computations.
  - ☣️ Choosing abbreviate forms like `CET`, which stands for `Central European
    Time` will behave differently on `PostgreSQL` and `MySQL`. Don't assume
    uniform behavior, or even a _rational_ one.
- ⚠️ Pay attention to the type of the `datetime` column you're working with. For
  example, in Postgres, a `datetime` can be one of the following types:
  1. `timestamp with time zone`
  2. `timestamp without time zone`
  In the first case, you don't need to supply a conversion hash because postgres
  knows how to convert it to the desired time zone. However, if you do the same
  for the second case, you might get surprises, especially if your Postgres
  installation's default timezone is not `UTC`.
- ⚠️ SQLite is not supported.
- 🚨 Always test against your setup 🚨

## Unions

```ruby
(t.where(t[:name].eq('str')) + t.where(t[:name].eq('test'))).to_sql
# => (SELECT * FROM my_table WHERE name='str') UNION (SELECT * FROM my_table WHERE name='test')
```

## Case clause

Arel-extensions allows to use functions on case clause

```ruby
t[:name].when("smith").then(1).when("doe").then(2).else(0).sum.to_sql
# => SUM(CASE "my_table"."name" WHEN 'smith' THEN 1 WHEN 'doe' THEN 2 ELSE 0 END)
```

## Cast Function

Arel-extensions allows to cast type on constants and attributes

```ruby
t[:id].cast('char').to_sql
# => CAST("my_table"."id" AS char)
```


## Stored Procedures and User-defined functions

To optimize queries, some classical functions are defined in databases missing any alternative native functions.
Examples :
- FIND_IN_SET

## BULK INSERT / UPSERT

Arel Extensions improves InsertManager by adding bulk_insert method, which allows to insert multiple rows in one insert.


```
@cols = ['id', 'name', 'comments', 'created_at']
@data = [
   	[23, 'name1', "sdfdsfdsfsdf", '2016-01-01'],
   	[25, 'name2', "sdfds234sfsdf", '2016-01-01']
]

insert_manager = Arel::InsertManager.new(User).into(User.arel_table)
insert_manager.bulk_insert(@cols, @data)
User.connection.execute(insert_manager.to_sql)
```

## New Arel Functions

<table class="tg arel-functions" style="font-size:80%">
  <thead>
  <tr>
    <th></th>
    <th class="tg-by3v">Function / Example<br/>ToSql</th>
    <th class="tg-pjz5">MySQL / MariaDB</th>
    <th class="tg-pjz5">PostgreSQL</th>
    <th class="tg-pjz5">SQLite</th>
    <th class="tg-pjz5">Oracle</th>
    <th class="tg-pjz5">MS SQL</th>
    <th class="tg-pjz5">DB2<br/>(not tested on real DB)</th>
  </tr>
  </thead>
  <tbody>
  <tr>
    <th class="tg-82sq" rowspan="7"><div>Number functions</div></th>
    <td class="tg-yw4l">ABS<br>column.abs<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">CEIL<br>column.ceil</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">CASE + CAST</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">CEILING()</td>
    <td class="tg-j6lv">CEILING()</td>
  </tr>
  <tr>
    <td class="tg-yw4l">FLOOR<br>column.floor</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">CASE + CAST</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">POSIX FORMATTING<br>column.format_number("$ %7.2f","en_US")</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ko">not implemented</td>
  </tr>
  <tr>
    <td class="tg-yw4l">RAND<br>Arel.rand</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">RANDOM()</td>
    <td class="tg-j6lv">dbms_random.value()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">ROUND<br>column.round(precision = 0)</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">SUM / AVG / MIN / MAX + x<br>column.sum + 42</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <th class="tg-ffjm" rowspan="17"><div>String functions</div></th>
    <td class="tg-yw4l">CONCAT<br>column + "string"</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv"> ||</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">+</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">FIND_IN_SET<br>column &amp; ("l")</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-orpl">Ruby function</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">ILIKE (in Arel6)<br/>column.imatches('%pattern')</td>
    <td class="tg-j6lv">LOWER() LIKE LOWER()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">LOWER() LIKE LOWER()</td>
    <td class="tg-j6lv">LOWER() LIKE LOWER()</td>
    <td class="tg-j6lv">LOWER() LIKE LOWER()</td>
  </tr>
  <tr>
    <td class="tg-yw4l">LENGTH<br>column.length</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">LEN()</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">LOCATE<br>column.locate("string")</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">INSTR() or Ruby function</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">CHARINDEX()</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">Matching Accent/Case Insensitive<br>column.ai_imatches('blah')</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">unaccent required</td>
    <td class="tg-j6lv">not supported</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">?</td>
  </tr>
  <tr>
    <td class="tg-yw4l">Matching Accent Insensitive<br>column.ai_matches('blah')</td>
    <td class="ok">not supported</td>
    <td class="tg-j6lv">not supported</td>
    <td class="tg-j6lv">not supported</td>
    <td class="ok">not supported</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">?</td>
  </tr>
  <tr>
    <td class="tg-yw4l">Matching Case Insensitive<br>column.imatches('blah')</td>
    <td class="ok">not supported</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">?</td>
  </tr>
  <tr>
    <td class="tg-yw4l">Matching Accent/Case Sensitive<br>column.smatches('blah')</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">not supported</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">✔</td>
    <td class="tg-j6lv">?</td>
  </tr>
  <tr>
    <td class="tg-yw4l">NOT_REGEXP<br>column != "pattern"</td>
    <td class="ok">✔</td>
    <td class="ok">✔<br></td>
    <td class="tg-3oug">require pcre.so</td>
    <td class="tg-j6lv">NOT REGEXP_LIKE </td>
    <td class="tg-j6lv">NOT LIKE</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">REGEXP<br>column =~ "pattern"<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-3oug">require pcre.so</td>
    <td class="tg-j6lv">REGEXP_LIKE</td>
    <td class="tg-j6lv">LIKE</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">REPLACE<br>column.replace("s","X")</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">SOUNDEX<br>column.soundex</td>
    <td class="ok">✔</td>
    <td class="tg-3oug">require fuzzystrmatch</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">SUBSTRING<br/>column[1..2]<br/>column.substring(1)<br/>column.substring(1, 1)</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">SUBSTR()</td>
    <td class="tg-j6lv">SUBSTR()</td>
    <td class="tg-j6lv">SUBSTR()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">TRIM (leading)<br>column.trim("LEADING","M")</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">LTRIM()</td>
    <td class="tg-j6lv">LTRIM()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">LTRIM()</td>
  </tr>
  <tr>
    <td class="tg-yw4l">TRIM (trailing)<br>column.trim("TRAILING","g")</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">RTRIM()</td>
    <td class="tg-j6lv">RTRIM()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">Rtrim()</td>
  </tr>
  <tr>
    <td class="tg-yw4l">TRIM (both)<br>column.trim("BOTH","e")</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">TRIM()<br></td>
    <td class="tg-j6lv">TRIM()</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">LTRIM(RTRIM())</td>
    <td class="tg-j6lv">TRIM()</td>
  </tr>
  <tr>
    <th class="tg-4rp9" rowspan="6"><div>Date functions</div></th>
    <td class="tg-yw4l">DATEADD<br>column + 2.year<br></td>
    <td class="tg-j6lv">DATE_ADD()<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">+</td>
  </tr>
  <tr>
    <td class="tg-yw4l">DATEDIFF<br>column - date<br></td>
    <td class="tg-j6lv">DATEDIFF()<br></td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">JULIANDAY() - JULIANDAY()</td>
    <td class="tg-j6lv"> -</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">DAY()</td>
  </tr>
  <tr>
    <td class="tg-yw4l">DAY<br>column.day<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">STRFTIME()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">MONTH<br>column.month<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">STRFTIME()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">WEEK<br>column.week</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">STRFTIME()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">YEAR<br>column.year</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">STRFTIME()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <th class="tg-72dn" rowspan="13"><div>Comparators functions</div></th>
    <td class="tg-yw4l">BLANK<br>column.blank<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">COALESCE<br>column.coalesce(var)</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">COALESCE_BLANK<br>column.coalesce_blank(var)</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">IF_PRESENT</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">ISNULL<br>column.isnull()</td>
    <td class="tg-j6lv">IFNULL()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="tg-j6lv">NVC()</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">NOT_BLANK<br>column.not_blank<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">PRESENT<br>column.present<br>alias to NOT_BLANK<br></td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">==<br>column == integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">!=<br>column != integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">&gt;<br>column &gt; integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">&gt;=<br>column &gt;= integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">&lt; <br>column &lt; integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">&lt;=<br>column &lt;= integer</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <th class="tg-9hbo" rowspan="2"><div>Boolean <br/> functions</div></th>
    <td class="tg-yw4l">OR ( ⋁ )<br>column.eq(var).⋁(column.eq(var))</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <td class="tg-yw4l">AND ( ⋀ )<br>column.eq(var).⋀(column.eq(var))</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  <tr>
    <th class="bulk_insert" rowspan="1"><div>Bulk <br/> Insert</div></th>
    <td class="tg-yw4l">insert_manager.bulk_insert(@cols, @data)</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>  <tr>
    <th class="set_operators" rowspan="1"><div>Set <br/> Operators</div></th>
    <td class="tg-yw4l">UNION (+)<br/>query + query</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>  <tr>
    <th class="set_operators" rowspan="1"><div>Set <br/> Operators</div></th>
    <td class="tg-yw4l">UNION ALL<br/>query.union_all(query)</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
    <td class="ok">✔</td>
  </tr>
  </tbody>
</table>

## Version Compatibility

<table>
  <tr><th>Ruby</th> <th>Rails</th>    <th>Arel Extensions</th></tr>
  <tr><td>3.1</td>  <td>6.1</td>      <td>2</td></tr>
  <tr><td>3.0</td>  <td>6.1</td>      <td>2</td></tr>
  <tr><td>2.7</td>  <td>6.1, 6.0</td> <td>2</td></tr>
  <tr><td>2.5</td>  <td>6.1, 6.0</td> <td>2</td></tr>
  <tr><td>2.5</td>  <td>5.2</td>      <td>1</td></tr>
</table>

## Development

Let's say you want to develop/test for `ruby 2.7.5` and `rails 5.2`.

You will need to fix your ruby version:

```bash
rbenv install 2.7.5
rbenv local 2.7.5
```

Fix your gemfiles:

```bash
bundle config set --local gemfile ./gemfiles/rails6.gemfile
```

Install dependencies:
```bash
bundle install
```

Develop, then test:

```bash
bundle exec rake test:to_sql
```

Refer to the [Version Compatibility](#version-compatibility) section to correctly
set your gemfile.
