![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        

# Rebuild All Indexes Across All Databases And Tables With SQL
**Post Date: November 14, 2015**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process

<p>You can put this into a Maintenance Job by it's self. It creates two temporary tables. One is a master list of all databases, types of objects, indexes, type of indexes etc. This is used when building a second temporary table, and all the work is pretty much built off the second table. The first table is there if you want to build other processes, or reporting against it; then you'll have no problem. This over all logic is intended for future growth. This does the normal thing by checking all databases, tables, and indexes, and divides them into two distinct categories. Those that are above 30% fragmentation and those that are below 30%. Those above will have their indexes rebuilt. Those below will have their indexes reorganized. That's it.</p>      


## SQL-Logic
```SQL
use master;
set nocount on
 
if object_id('tempdb..#fragmented_indexes') is not null
    drop table  #fragmented_indexes
create table    #fragmented_indexes
(
    [database]          varchar(255) null
,   [table_name]        varchar(255) null
,   [index_id]          int
,   [index]             varchar(255) null
,   [fragmentation]     float null
,   [index_type]        varchar(255) null
,   [recommend]         varchar(255)
)
 
declare @dbname         varchar(1000)
declare @get_index_info nvarchar(4000)
declare dbcursor        cursor for
    select
        name
    from
        sys.databases
    where
        name not in ('master', 'model', 'msdb', 'tempdb')
open        dbcursor 
fetch next from dbcursor into @dbname
while @@fetch_status = 0
    begin
        set @get_index_info = '
            use [' + @dbname + '];
            if exists (select compatibility_level from sys.databases where name  = N'''+ @dbname +'''   and compatibility_level >= 90)
            begin
                insert into #fragmented_indexes 
                select 
                    [database]          = upper(db_name())
                ,   [table_name]        = upper(object_name(sddips.object_id))
                ,   [index_id]          = sddips.index_id
                ,   [index]             = si.name
                ,   [fragmentation]     = [avg_fragmentation_in_percent]
                ,   [index_type]        = index_type_desc
                ,   [recommend]         =   case
                                                when [avg_fragmentation_in_percent] > 30 then ''REBUILD''
                                                when [avg_fragmentation_in_percent] < 30 then ''REORGANIZE''
                                                else ''none''
                                            end
                from 
                    sys.dm_db_index_physical_stats (db_id(), null, null, null, null) sddips 
                    join    sys.indexes si on sddips.object_id = si.object_id 
                    and     sddips.index_id = si.index_id 
                where
                    si.index_id <> 0 
                    and     [avg_fragmentation_in_percent] <> 0
            end;'
            exec sp_executesql @get_index_info
            fetch next from dbcursor
        into @dbname
    end
close       dbcursor
deallocate  dbcursor
 
 
if object_id('tempdb..#fragmented_indexes_per_database') is not null
    drop table #fragmented_indexes_per_database
create table #fragmented_indexes_per_database
(
    [database]      varchar(255)
,   [table_name]    varchar(255)
,   [num_of_index]  int
,   [fragmentation] float null
)
insert into #fragmented_indexes_per_database
select
    [database]
,   [table_name]
,   count([table_name])
,   max([fragmentation])
from
    #fragmented_indexes
group by
    [database], [table_name]
having
    (count([table_name]) > 1)
order by
    [database], [table_name] asc
 
declare @rebuild_indexes        varchar(max)
set     @rebuild_indexes        = ''
select  @rebuild_indexes        = @rebuild_indexes +
'use [' + [database] + '];' + char(10) + 
case
    when [fragmentation] > 30 then 'alter index all on [' + [table_name] + '] rebuild with (online = on);' + char(10)
    when [fragmentation] < 30 then 'alter index all on [' + [table_name] + '] reorganize;' + char(10)
end
from
    #fragmented_indexes_per_database fipr 
    join sys.databases sd on fipr.[database] = sd.[name]
    join sys.database_mirroring sdm on sd.database_id = sdm.database_id
where
    sd.name not in ('tempdb')
    and sd.state_desc = 'online'
    and sdm.mirroring_role_desc is null
    or  sdm.mirroring_role_desc != 'mirror'
order by
    [database], [fragmentation] asc
 
exec (@rebuild_indexes)
 
 
drop table #fragmented_indexes
drop table #fragmented_indexes_per_database
```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

   
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

