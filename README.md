# bul-traject-analysis

An annotated analysis of [Brown's implementation of Traject](https://github.com/Brown-University-Library/bul-traject), created to learn more about the set of tools.

## File manifest

### Primary

- `config.rb`: Main Traject configuration file
- `test_config.rb`: Alternate Traject configuration file ("Bare config for testing new mappings")

### Secondary

- `lib/bul_format.rb`: Analyze fixed fields to provide a format code (`class Format`)
- `lib/bul_macros.rb`: Define macros – custom functions – for complex indexing and data processing (`module BulMacros`)
- `lib/bul_utils.rb`: Define additional helper functions
- `lib/translation_maps/buildings.yaml`: Map location code to human-readable label
- `lib/translation_maps/format.rb`: Map format code to human-readable label

### Development

- `spec/*`: Functional specifications for use with a Ruby automated testing tool
- `spec/fixtures/*`: Sample MARC records for use in testing

### Administrative

- `Gemfile`, `Gemfile.lock`: Configurations for the RubyGems package manager
- `LICENSE`: Software license for the package
- `README.md`: This README File

## bul-traject
This project transforms MARC records into Solr documents using the [Traject](https://github.com/traject-project/traject) tools developed by [Bill Dueber](https://github.com/billdueber/) and [Jonathan Rochkind](https://github.com/jrochkind).

Run as:
```
traject -c config.rb -u http://localhost:8081/solr/blacklight-core /full/path/to/marcfile.mrc

curl http://localhost:8081/solr/blacklight-core/update?commit=true
```
