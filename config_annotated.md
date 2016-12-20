# config.rb

Detailed documentation for `config.rb`. For further information, see the [Traject project documentation](https://github.com/traject/traject).

Check whether the Java implementation of Ruby is being used; if so, import an additional requirement from the default Traject installation.

```ruby
#
#Brown MARC to Solr indexing
#Uses traject: https://github.com/traject-project/traject
#

#Check if we are using jruby and store.
is_jruby = RUBY_ENGINE == 'jruby'
if is_jruby
  require 'traject/marc4j_reader'
end
```

Import the files containing the macros supplied by Traject.

```ruby
#Translation maps.
# './lib/translation_maps/'
$:.unshift  "#{File.dirname(__FILE__)}/lib"

require 'traject/macros/marc21_semantics'
extend  Traject::Macros::Marc21Semantics

require 'traject/macros/marc_format_classifier'
extend Traject::Macros::MarcFormats
```

Import the files under `/lib` containing Brown's custom macros and helper functions, including the module called `BulMacros`.

```ruby
#local macros
require 'bul_macros'
extend BulMacros

#local utils
require 'bul_utils'
require 'bul_format'
```

Define settings that will be used by the system when running this indexing process.

```ruby
# Setup
settings do
  store "log.batch_progress", 10_000
  provide "solr.url", ENV['SOLR_URL']
  #Use Marc4JReader and solrj writer when available.
  if is_jruby
    provide "reader_class_name", "Traject::Marc4JReader"
    provide "marc4j_reader.source_encoding", "UTF-8"
    provide "solrj_writer.commit_on_close", "true"
    # Use more threads on local box.
    if ENV['TRAJECT_ENV'] == "devbox"
      provide 'processing_thread_pool', 8
    else
      provide 'processing_thread_pool', 3
    end
  end
end

logger.info RUBY_DESCRIPTION
```

Every function and indexing rule defined below will apply to each processed record, if applicable.

Skip the indexing of records identified as suppressed, using a custom function, `suppressed`, defined in `lib/bul_utils`.

Check whether the item described in each record is marked as being available online, using another custom function, `is_online`, also defined in `lib/bul_utils`, and store the result of that check in memory for subsequent use.

```ruby
each_record do |rec, context|
  if suppressed(rec) == true
    context.skip!("Skipping suppressed record")
  end
  #We will use this twice so hang on to it.
  context.clipboard[:is_online] = is_online(rec)
end
```

The `to_field` instruction identifies a Solr field to be populated, using a macro referenced by name, or using a block of code that is specified in-line. Here, populate the Solr field `id` by using the macro `record_id`, which is defined in `BulMacros`.

```ruby
#Brown record id
to_field "id", record_id
```

Similarly, populate the Solr field `updated_dt` using the macro `updated_date`, which is also defined in `BulMacros`.

```ruby
#Brown last updated date
to_field "updated_dt", updated_date
```

`extract_marc` is the primary built-in Traject macro, allowing data to be extracted from each occurrence of the specified MARC field(s), limited to the contents of the identified subfields. A colon separates each instance of a field/subfield combination from which data should be extracted. A few options can be invoked each time `extract_marc` is used to override the defaults, if necessary.

For `isbn_t`, extract data from the 020 $a and the 020 $z.

For `issn_t`, extract data from the the specified fields/subfields.

For `oclc_t`, apply the `oclcnum` macro from Traject's `Marc21Semantics` module.

```ruby
#Identifiers
to_field 'isbn_t', extract_marc('020a:020z')
to_field 'issn_t', extract_marc("022a:022l:022y:773x:774x:776x", :separator => nil)
to_field 'oclc_t', oclcnum('001:035a:035z')
```

Here, extract data from the following field/subfield combinations defined on each line into `title_t`. When set to "true", `trim_punctuation` implements a [built-in routine](https://github.com/traject/traject/blob/c23d2045d49ca60be6b70a19e471e739e86b1d51/lib/traject/macros/marc21.rb#L216-L246) for removing certain trailing or leading punctuation.

From Traject's [documentation on subfield concatenation](http://www.rubydoc.info/gems/traject/Traject/MarcExtractor):

"For a spec including multiple subfield codes, multiple subfields from the same MARC field will be concatenated into one string separated by spaces. You can turn off this concatenation and leave individual subfields in seperate strings by setting the `separator` option to nil. However, the default is different for specifications with only a single subfield, these are by default kept separated."

```ruby
# Title fields
to_field 'title_t', extract_marc(%w(
  100tflnp
  110tflnp
  111tfklpsv
  130adfklmnoprst
  210ab
  222ab
  240adfklmnoprs
  242abnp
  246abnp
  247abnp
  505t
  700fklmnoprstv
  710fklmorstv
  711fklpt
  730adfklmnoprstv
  740ap
  ),
  :trim_punctuation => true
)
```

For `title_display`, extract data from the specified subfields in only the first occurrence of the 245.

Do the same in `title_vern_display`, except from the corresponding 880 field. By default, `alternate_script` is "true" and will extract data from both the specified regular field and the linked 880 field where available. When set to "false", `alternate_script` will exclude linked 880 fields, and when set to "only", it will include the 880 field but not its equivalent.

```ruby
to_field 'title_display', extract_marc('245abfgknp', :first=>true, :trim_punctuation => true)
to_field 'title_vern_display', extract_marc('245abfgknp', :alternate_script=>:only, :trim_punctuation => true, :first=>true)
```

Extract data for `title_series_t` in a manner similar to that for `title_t`.

For `title_sort`, use the default Traject macro `marc_sortable_title`, which generates [a version of the title](https://github.com/traject/traject/blob/1da24e2f0efeaa3386f72eeb87053b588561f501/lib/traject/macros/marc21_semantics.rb#L91-L118) without any non-filing characters.

```ruby
to_field 'title_series_t', extract_marc(%w(
  400flnptv
  410flnptv
  411fklnptv
  440ap
  490a
  800abcdflnpqt
  810tflnp
  811tfklpsv
  830adfklmnoprstv
  ),
  :trim_punctuation => true
)
to_field "title_sort", marc_sortable_title
```

Apply the custom macros `get_uniform_titles_info`, `get_uniform_title_author_info`, and `get_uniform_related_works_info` to extract data into the specified Solr fields.

```ruby
# Uniform Titles
to_field 'uniform_titles_display' do |record, accumulator, context|
  info = get_uniform_titles_info(record)
  if !info.nil?
    accumulator << info
  end
end
to_field 'new_uniform_title_author_display' do |record, accumulator, context|
  info = get_uniform_title_author_info(record)
  if !info.nil?
    accumulator << info
  end
end
to_field 'uniform_related_works_display' do |record, accumulator, context|
  info = get_uniform_related_works_info(record)
  if !info.nil?
    accumulator << info
  end
end
```

Extract data from the specified field/subfield combinations.

```ruby
# Author fields
to_field "author_display", extract_marc("100abcdq:110abcd:111abcd", :first=>true, :trim_punctuation => true)
to_field "author_vern_display", extract_marc('100abcdq:110abcd:111abcd', :alternate_script=>:only, :trim_punctuation => true, :first=>true)
to_field "author_addl_display", extract_marc('700abcd:710ab:711ab', :trim_punctuation => true)
to_field "author_t", extract_marc('100abcdq:110abcd:111abcdeq', :trim_punctuation => true)
to_field 'author_addl_t', extract_marc("700abcdq:710abcd:711abcdeq:810abc:811aqdce")
```

Extract data from the specified field/subfield combinations.

Apply the custom macros `toc_display` and `toc_970_display` and the Traject default macro `marc_publication_date`.

```ruby
#
# - Publication details fields
#
to_field "published_display", extract_marc("260a", :trim_punctuation=>true)
to_field "published_vern_display",  extract_marc("260a", :alternate_script => :only)
#Display physical information.
to_field 'physical_display', extract_marc('300abcefg:530abcd')
to_field "abstract_display", extract_marc("520a", :first=>true)
to_field "toc_display" do |record, accumulator, context|
  info = get_toc_505_info(record)
  if !info.nil?
    accumulator << info
  end
end
to_field "toc_970_display" do |record, accumulator, context|
  info = get_toc_970_info(record)
  if !info.nil?
    accumulator << info
  end
end

to_field "pub_date", marc_publication_date
```

Extract data from subfields of the same field into two different Solr fields.

```ruby
#URL Fields - these will have to be custom, most likely.
to_field "url_fulltext_display", extract_marc("856u")
to_field "url_suppl_display", extract_marc("856z")
```

For `online_b`, recall the results of the `is_online` check that were held in memory.

```ruby
#Online true/false
to_field "online_b" do |record, accumulator, context|
  accumulator << context.clipboard[:is_online]
end
```

For `access_facet`, store a human-readable value based on the results of the `is_online` check.

```ruby
#Access facet
to_field "access_facet" do |record, accumulator, context|
  online = context.clipboard[:is_online]
  if online == true
    val = "Online"
  else
    val = "At the library"
  end
  accumulator << val
end
```

For `format`, Brown's custom class `Format` is used to analyze the fixed fields of the record and provide an appropriate format code. This code is then compared with a Traject construct called a "translation map" -- here also called `format` -- in order to retrieve a human-readable label for display. So:

1. Custom `Format` class, defined in `lib/bul_format`, evaluates the record and returns a format code
2. The code is run through the `format` translation map, defined in `lib/translation_maps/format`
3. The result is stored in the Solr field called `format`

```ruby
#
# - Facet fields
#

#Custom logic for format
to_field 'format' do |record, accumulator|
  tmap = Traject::TranslationMap.new('format')
  bf = Format.new(record)
  value = tmap[bf.code]
  accumulator << value
end
```

`author_facet` is a custom macro.

For `language_facet`, use the default Traject macro `marc_languages` on data extracted from the 008 (bytes 35-37), 041 $a, and 041 $d.

```ruby
#Author - local macro
to_field "author_facet", author_facet
#Language
to_field 'language_facet', marc_languages("008[35-37]:041a:041d:041e:041j")
```

For `building_facet`, extract data from the 945 $l and use the function `map_code_to_building`, defined in `lib/bul_utils`.

```ruby
#Buildings - Unique list of 945s sf l processed through the translation map.
to_field "building_facet", extract_marc('945l') do |record, acc|
  acc.map!{|code| map_code_to_building(code)}.uniq!
end
```

For `location_code_t` and `topic_facet`, extract data from the specified field/subfield combinations.

For `region_facet`, apply the `marc_geo_facet` macro from Traject's `Marc21Semantics` module.

```ruby
# There can be 0-N location codes per BIB record
# because the location code is at the item level.
to_field "location_code_t", extract_marc('945l', :trim_punctuation => true)

to_field "region_facet", marc_geo_facet
to_field "topic_facet", extract_marc("650a:690a", :trim_punctuation => true)
```

Extract data from the specified field/subfield combinations.

```ruby
#
# - Search fields
#

#Subject search.  See: https://github.com/billdueber/ht_traject/blob/master/indexers/common.rb
to_field "subject_t", extract_marc(%w(
  600a  600abcdefghjklmnopqrstuvxyz
  610a  610abcdefghklmnoprstuvxyz
  611a  611acdefghjklnpqstuvxyz
  630a  630adefghklmnoprstvxyz
  648a  648avxyz
  650a  650abcdevxyz
  651a  651aevxyz
  653a  654abevyz
  654a  655abvxyz
  655a  656akvxyz
  656a  657avxyz
  657a  658ab
  658a  662abcdefgh
  690a   690abcdevxyz
  ), :trim_punctuation=>true)

#Callnumber
to_field "callnumber_t", extract_marc("050ab:090ab:091ab:092ab:096ab:099ab", :trim_punctuation => true)
```

For `text`, use the default Traject macro `extract_all_marc_values` to populate a single Solr field with a set of data across the specified range of MARC fields. The results of the `get_toc_970_indexing` macro are then appended to the field.

```ruby
#Text - for search
to_field "text", extract_all_marc_values(:from=>'090', :to=>'900')
to_field "text" do |record, accumulator, context|
    get_toc_970_indexing(record, accumulator)
end
```

For `marc_display`, serialize the input record as JSON.

```ruby
to_field "marc_display", serialized_marc(:format => "json", :allow_oversized => true)
```

For `bookplate_code_facet`, extract data from the 945 $f.

```ruby
# There can be 0-N bookplate codes per BIB record
# because the bookplate info is at the item level.
# I am using "_facet" because that is a string,
# indexed, multivalue field and I don't want to alter
# Solr's config to add a new field type just yet.
to_field "bookplate_code_facet", extract_marc("945f")
```
