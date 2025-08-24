# gepub  

[![GitHub Actions Status](https://github.com/skoji/gepub/workflows/Test/badge.svg)](https://github.com/skoji/gepub/actions?query=workflow%3ATest)
[![Gem Version](https://badge.fury.io/rb/gepub.svg?icon=si%3Arubygems)](https://badge.fury.io/rb/gepub)

* https://rubydoc.info/github/skoji/gepub

## DESCRIPTION:

a generic EPUB parser/generator library.

## FEATURES/PROBLEMS:

* GEPUB::Book provides functionality to create EPUB files and parse EPUB files
* Handle every metadata in EPUB2/EPUB3.

* See [issues](https://github.com/skoji/gepub/issues/) for known problems.

If you are using GEPUB::Builder and do not like its behavior (e.g., GEPUB::Builder evaluates the block as inside the Builder instance),  consider using GEPUB::Book directly.

## SYNOPSIS:

### Example

```ruby
require 'rubygems'
require 'gepub'

book = GEPUB::Book.new
book.primary_identifier('http://example.jp/bookid_in_url', 'BookID', 'URL')
book.language = 'ja'

book.add_title 'GEPUBサンプル文書', 
               title_type: GEPUB::TITLE_TYPE::MAIN,
               lang: 'ja',
               file_as: 'GEPUB Sample Book',
               display_seq: 1,
               alternates: {
                       'en' => 'GEPUB Sample Book (Japanese)',
                       'el' => 'GEPUB δείγμα (Ιαπωνικά)',
                       'th' => 'GEPUB ตัวอย่าง (ญี่ปุ่น)' }
               
# you can do the same thing using method chain
book.add_title('これはあくまでサンプルです', title_type: GEPUB::TITLE_TYPE::SUBTITLE).display_seq(1).add_alternates('en' => 'this book is just a sample.')

# use arguments
book.add_creator '小嶋智', 
                 display_seq:1, 
                 alternates: { 'en' => 'KOJIMA Satoshi' } 
book.add_contributor '電書部',
                     display_seq: 1,
                     alternates: {'en' => 'Denshobu'}
book.add_contributor 'アサガヤデンショ',
                     display_seq: 2, 
                     alternates: {'en' => 'Asagaya Densho'}
# you can also use method chain
book.add_contributor('湘南電書鼎談').display_seq(3).add_alternates('en' => 'Shonan Densho Teidan')
book.add_contributor('電子雑誌トルタル').display_seq(4).add_alternates('en' => 'eMagazine Torutaru')

imgfile = File.join(File.dirname(__FILE__),  'image1.jpg')
File.open(imgfile) do
  |io|
  book.add_item('img/image1.jpg',content: io).cover_image
end

# within ordered block, add_item will be added to spine.
book.ordered {
  book.add_item('text/cover.xhtml',
                content: StringIO.new(<<-COVER)).landmark(type: 'cover', title: 'cover page')
                <html xmlns="http://www.w3.org/1999/xhtml">
                <head>
                  <title>cover page</title>
                </head>
                <body>
                <h1>The Book</h1>
                <img src="../img/image1.jpg" />
                </body></html>
  COVER
  book.add_item('text/chap1.xhtml').add_content(StringIO.new(<<-CHAP_ONE)).toc_text('Chapter 1').landmark(type: 'bodymatter', title: '本文')
  <html xmlns="http://www.w3.org/1999/xhtml">
  <head><title>c1</title></head>
  <body><p>the first page</p></body></html>
  CHAP_ONE
  book.add_item('text/chap1-1.xhtml').add_content(StringIO.new(<<-SEC_ONE_ONE)) # do not appear on table of contents
  <html xmlns="http://www.w3.org/1999/xhtml">
  <head><title>c2</title></head>
  <body><p>the second page</p></body></html>
  SEC_ONE_ONE
  book.add_item('text/chap2.xhtml').add_content(StringIO.new(<<-CHAP_TWO)).toc_text('Chapter 2')
  <html xmlns="http://www.w3.org/1999/xhtml">
  <head><title>c3</title></head>
  <body><p>the third page</p></body></html>
  CHAP_TWO
  # to add nav file:
  # book.add_item('path/to/nav').add_content(nav_html_content).add_property('nav')
}
epubname = File.join(File.dirname(__FILE__), 'example_test.epub')

# if you do not specify a nav document with add_item, 
# generate_epub will generate simple navigation text.
# auto-generated nav file will not appear on the spine.
book.generate_epub(epubname)
```
 * [examples in this repository](https://github.com/skoji/gepub/tree/main/examples/) 

### Parsing and Reading EPUB Files

GEPUB can also parse existing EPUB files to extract metadata, content, and structure.

#### Basic Parsing

```ruby
require 'gepub'

# Parse an EPUB file (works with file path or IO object)
book = GEPUB::Book.parse('path/to/your/book.epub')

# Or with an IO object
File.open('path/to/your/book.epub', 'rb') do |file|
  book = GEPUB::Book.parse(file)
end
```

#### Accessing Metadata

```ruby
# Basic metadata
puts book.title        # Book title
puts book.creator       # Primary author/creator
puts book.language      # Language code (e.g., "en-US")
puts book.identifier    # Unique identifier
puts book.publisher     # Publisher (may be nil)
puts book.date          # Publication date (may be nil)
puts book.description   # Description (may be nil)
puts book.subject       # Subject/topic (may be nil)
```

#### Accessing Book Structure and Content

```ruby
# Get all items (chapters, images, stylesheets, etc.)
book.items.each do |item_id, item|
  puts "#{item_id}: #{item.href} (#{item.media_type})"
end

# Get the reading order (spine)
book.spine_items.each_with_index do |item, index|
  puts "Chapter #{index + 1}: #{item.href}"
end

# Access content of a specific item
first_chapter = book.spine_items.first
content = first_chapter.content
puts "Content: #{content[0..200]}..."  # First 200 characters

# Access images and other resources
book.items.each do |item_id, item|
  if item.media_type.start_with?('image/')
    puts "Image: #{item.href} (#{item.content.length} bytes)"
  end
end
```

#### Working with Navigation

```ruby
# Access the navigation document (EPUB3) or NCX file (EPUB2)
nav_item = book.items.values.find { |item| 
  item.media_type == 'application/xhtml+xml' && 
  (item.properties&.include?('nav') || item.href.include?('nav'))
}

if nav_item
  nav_content = nav_item.content
  puts "Navigation document: #{nav_item.href}"
end
```

#### Complete Example: Reading an EPUB from Start to Finish

```ruby
require 'gepub'

def read_epub(epub_path)
  # Parse the EPUB file
  book = GEPUB::Book.parse(epub_path)
  
  # Display basic information
  puts "=" * 50
  puts "EPUB Information"
  puts "=" * 50
  puts "Title: #{book.title}"
  puts "Author: #{book.creator}"
  puts "Language: #{book.language}"
  puts "Identifier: #{book.identifier}"
  puts "Publication Date: #{book.date}" if book.date
  puts
  
  # Show book structure
  puts "Book Structure (#{book.spine_items.length} chapters):"
  puts "-" * 30
  book.spine_items.each_with_index do |item, index|
    puts "#{index + 1}. #{item.href}"
  end
  puts
  
  # Show all resources
  puts "All Resources (#{book.items.length} items):"
  puts "-" * 30
  content_types = {}
  book.items.each do |item_id, item|
    content_types[item.media_type] ||= []
    content_types[item.media_type] << item.href
  end
  
  content_types.each do |media_type, files|
    puts "#{media_type}:"
    files.each { |file| puts "  - #{file}" }
  end
  puts
  
  # Read and display content from each chapter
  puts "Chapter Contents:"
  puts "-" * 30
  book.spine_items.each_with_index do |item, index|
    puts "Chapter #{index + 1}: #{item.href}"
    content = item.content
    
    # Extract text content (basic HTML stripping for display)
    text_content = content.gsub(/<[^>]*>/, '').strip
    preview = text_content[0..200].gsub(/\s+/, ' ')
    puts "  Preview: #{preview}#{'...' if text_content.length > 200}"
    puts "  Full length: #{content.length} characters"
    puts
  end
end

# Usage
read_epub('path/to/your/book.epub')
```

#### Error Handling

```ruby
begin
  book = GEPUB::Book.parse('path/to/book.epub')
  puts "Successfully parsed: #{book.title}"
rescue => e
  puts "Error parsing EPUB: #{e.message}"
end
```

#### Extracting Plain Text Content

For text processing, you might want to extract clean text from XHTML chapters:

```ruby
require 'nokogiri'

book = GEPUB::Book.parse('path/to/book.epub')

def extract_text(xhtml_content)
  # Parse HTML and extract text content
  doc = Nokogiri::HTML(xhtml_content)
  text = doc.css('body').text
  # Clean up whitespace
  text.gsub(/\s+/, ' ').strip
end

# Extract text from all chapters
book.spine_items.each_with_index do |item, index|
  if item.media_type == 'application/xhtml+xml'
    text = extract_text(item.content)
    puts "Chapter #{index + 1}: #{text[0..100]}..." unless text.empty?
  end
end
```

#### Working with Different EPUB Versions

GEPUB automatically handles both EPUB2 and EPUB3 formats. The parsing interface remains the same, but some features may vary:

```ruby
book = GEPUB::Book.parse('path/to/book.epub')

# Check EPUB version
puts "EPUB Version: #{book.version}"

# EPUB3 books may have additional features
if book.version.to_f >= 3.0
  # Look for navigation document
  nav_item = book.items.values.find { |item| 
    item.properties&.include?('nav')
  }
  puts "Navigation document: #{nav_item.href}" if nav_item
end
```

## INSTALL:

* gem install gepub

## DONATE:

* Bitcoin Address: `1M69AwoxpgPZsp5KStLUEjP7so5dHVfDTH`
