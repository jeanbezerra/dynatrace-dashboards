

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