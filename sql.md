## Change Database Tables to InnoDB
_Run this SQL Query in PhpMyAdmin (changing_ `DatabaseName` _to the actual database name). It will output a list of commands to change MyISAM tables to InnoDB. Be sure that you show full texts so that each command is not truncated._

```
SET @DATABASE_NAME = 'DatabaseName';

SELECT CONCAT('ALTER TABLE `', table_name, '` ENGINE=InnoDB;') AS sql_statements
FROM information_schema.tables AS tb
WHERE table_schema = @DATABASE_NAME
AND `ENGINE` = 'MyISAM'
AND `TABLE_TYPE` = 'BASE TABLE'
ORDER BY table_name DESC;
```

## Fixing Modified Dates
_If modified dates ended up earlier than published dates, these queries can quickly make them match the published dates.  **As always, be sure to update the prefix.**_

_Provide a CSV list of Post IDs to edit:_
```
UPDATE wp_posts
SET post_modified = post_date,
    post_modified_gmt = post_date_gmt
WHERE post_status = 'publish'
  AND ID IN (1, 2, 3, 4, 5, 6);
```
_Updates all posts where the modified date is earlier than the published date. (Note that scheduled posts often have the modified date slightly earlier, so this will change those too.)[UNTESTED]:_
```
UPDATE wp_posts
SET post_modified = post_date,
    post_modified_gmt = post_date_gmt
WHERE post_status = 'publish'
  AND post_modified < post_date;
```
