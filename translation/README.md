Direnvを使った構成の例は次の通りです。

```shell
if has guix
then
    use guix
else
    PATH_add "$HOME/src/po4a"
    path_add PERL5LIB "$HOME/src/po4a/lib"
    layout perl
    path_add C_INCLUDE_PATH /usr/local/include
    cpanm --notest YAML::Tiny Syntax::Keyword::Try
fi
```
