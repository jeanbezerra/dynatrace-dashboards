
```dql
fetch dt.entity.host, from:-30d
| expand tags, alias:tag_string
| parse tag_string,
    """(('['LD:tag_context ']' LD:tag_key (!<<'\\' ':') LD:tag_value)|(LD:tag_key (!<<'\\' ':') LD:tag_value)|LD:tag_key)"""
| fields
    Host = entity.name,
    TagCompleta = tag_string,
    Contexto = tag_context,
    Chave = tag_key,
    Valor = tag_value
| sort Chave asc, Valor asc, Host asc
```


```dql
fetch dt.entity.host, from:-30d
| expand tags, alias:tag_string
| parse tag_string,
    """(('['LD:tag_context ']' LD:tag_key (!<<'\\' ':') LD:tag_value)|(LD:tag_key (!<<'\\' ':') LD:tag_value)|LD:tag_key)"""
| filter
    lower(tag_key) == "environment"
    or lower(tag_key) == "ambiente"
    or lower(tag_key) == "env"
| filter isNotNull(tag_value)
| fields Environment = tag_value
| dedup Environment
| sort Environment asc
```


## Variável Host dependente de Environment

```dql
fetch dt.entity.host, from:-30d
| expand tags, alias:tag_string
| parse tag_string,
    """(('['LD:tag_context ']' LD:tag_key (!<<'\\' ':') LD:tag_value)|(LD:tag_key (!<<'\\' ':') LD:tag_value)|LD:tag_key)"""
| filter
    (
      lower(tag_key) == "environment"
      or lower(tag_key) == "ambiente"
      or lower(tag_key) == "env"
    )
    and tag_value == $Environment
| fields Host = entity.name
| dedup Host
| sort Host asc
```