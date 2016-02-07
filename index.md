
Some inline `code`

Generic multiline code
```
cmd1
*cmd2*
cmd3
__cmd4__
cmd5
```

Multiline code with syntax highlighting
```sql
select distinct
  m.cowboy_user as cowboy_name,
  lp.base_command as command
from
  user_map m,
  linux_process lp
where
  trim(lp.owner)=trim(m.okey_user)
order by
  cowboy_name desc;
```

Some __double underscores__ and **double asterisks** and \_\_escaped double underscores\_\_ and \*\*escaped double asterisks\*\*.

Some _single underscores_ and *single asterisks* and \_escaped single underscores\_ and \*escaped single asterisks\*.

Some symmetric ___triple underscores___ and ***triple asterisks*** and _**uaa-aau**_ and *__auu-uua__* and __*uua-auu*__ and **_aau-uaa_**
and *_*aua-aua*_* and _*_uau-uau_*_ 

Some asymmetric __*uua-uua__* and **_aau-aau**_ and _**uaa-uaa_** and *__auu-auu*__
