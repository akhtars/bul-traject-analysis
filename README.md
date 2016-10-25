# bul-traject-analysis

An annotated analysis of [Brown's implementation of Traject](https://github.com/Brown-University-Library/bul-traject), created to learn more about the set of tools.

## bul-traject
This project transforms MARC records into Solr documents using the [Traject](https://github.com/traject-project/traject) tools developed by [Bill Dueber](https://github.com/billdueber/) and [Jonathan Rochkind](https://github.com/jrochkind).

Run as:
```
traject -c config.rb -u http://localhost:8081/solr/blacklight-core /full/path/to/marcfile.mrc

curl http://localhost:8081/solr/blacklight-core/update?commit=true
```
