# BibTeX Parser

This is a [BibTeX](http://mirrors.ctan.org/biblio/bibtex/base/btxdoc.pdf) parser written in PHP.

[![Build Status](https://travis-ci.org/renanbr/bibtex-parser.svg?branch=master)](https://travis-ci.org/renanbr/bibtex-parser)

## Installation

```bash
composer require renanbr/bibtex-parser ^2@dev
```

[Read documentation for version 1](https://github.com/renanbr/bibtex-parser/blob/1.x/README.md)

## Usage

```php
use RenanBr\BibTexParser\Listener;
use RenanBr\BibTexParser\Parser;

require 'vendor/autoload.php';

$bibtex = <<<BIBTEX
@article{einstein1916relativity,
  title={Relativity: The Special and General Theory},
  author={Einstein, Albert},
  year={1916}
}
BIBTEX;

$parser = new Parser();          // Create a Parser
$listener = new Listener();      // Create and configure a Listener
$parser->addListener($listener); // Attach the Listener to the Parser
$parser->parseString($bibtex);   // or parseFile('/path/to/file.bib')
$entries = $listener->export();  // Get processed data from the Listener

print_r($entries);
```

This will output:

```
Array
(
    [0] => Array
        (
            [type] => article
            [citation-key] => einstein1916relativity
            [title] => Relativity: The Special and General Theory
            [author] => Einstein, Albert
            [year] => 1916
        )
)
```

## Vocabulary

BibTeX is all about "entry", "tag's name" and "tag's content".

> A BibTeX **entry** consists of the type (the word after @), a citation-key and a number of tags which define various characteristics of the specific BibTeX entry. (...) A BibTeX **tag** is specified by its **name** followed by an equals-sign and the **content**.

Source: http://www.bibtex.org/Format/

Note: This library considers "type" and "citation-key" as tags. This behavior can be change if you implement your own Listener (more info at the end of this document).

## Processors

This library contains three main parts:

- `Parser` class, responsible for detecting units inside a BibTeX input;
- `Listener` class, responsible for gathering units and transforming them into a list of entries;
- `Processor` classes, responsible for manipulating entries.

Despite we can't configure the `Parser`, you can append as many `Processor` as you want to the `Listener`. If you need more than this, considering implementing your own `Listener` (more info at the end of this document).

Before showing the available processors, get awareness that `Listener` provides, by default, these features:

- Found entries are reachable through `export()` method;
- [Tag content concatenation](http://www.bibtex.org/Format/);
- [Tag content abbreviation handling](http://www.bibtex.org/Format/);
- Publication's type is exposed as `type` tag;
- Citation key is exposed as `citation-key` tag;
- Original entry text is exposed as `_original` tag.

`Processor` is a [callable] that receives an entry as argument and returns a modified entry. You can append processors through `Listener::addProcessor()` before exporting the contents. This project is shipped with some useful processors.

### Tag name case

In BibTeX the tag's names aren't case-sensitive. This library exposes entries as array, but in PHP array keys are case-sensitive. To avoid this misunderstanding, you can force the tags' character case using `TagNameCaseProcessor`.

```php
use RenanBr\BibTexParser\Processor\TagNameCaseProcessor;

$listener->addProcessor(new TagNameCaseProcessor(CASE_UPPER)); // or CASE_LOWER
```

```bib
@article{
  title={BibTeX rocks}
}
```

```
Array
(
    [0] => Array
        (
            [TYPE] => article
            [TITLE] => BibTeX rocks
        )
)
```

### Authors and editors

BibTeX recognizes four parts of an author's name: First Von Last Jr. If you would like to parse the `author` and `editor` tags included in your entries, you can use the `NamesProcessor` class.

```php
use RenanBr\BibTexParser\Processor\NamesProcessor;

$listener->addProcessor(new NamesProcessor());
```

```bib
@article{
  title={Relativity: The Special and General Theory},
  author={Einstein, Albert}
}
```

```
Array
(
    [0] => Array
        (
            [type] => article
            [title] => Relativity: The Special and General Theory
            [author] => Array
                (
                    [0] => Array
                        (
                            [first] => Albert
                            [von] =>
                            [last] => Einstein
                            [jr] =>
                        )
                )
        )
)
```

### Keywords

The `keywords` tag contains a list of expressions represented as text, you might want to read them as an array instead.

```php
RenanBr\BibTexParser\Processor\KeywordsProcessor;

$listener->addProcessor(new KeywordsProcessor());
```

```bib
@misc{
  title={The End of Theory: The Data Deluge Makes the Scientific Method Obsolete},
  keywords={big data, data deluge, scientific method}
}
```

```
Array
(
    [0] => Array
        (
            [type] => misc
            [title] => The End of Theory: The Data Deluge Makes the Scientific Method Obsolete
            [keywords] => Array
                (
                    [0] => big data
                    [1] => data deluge
                    [2] => scientific method
                )
        )
)
```

### LaTeX to unicode

BibTeX files store LaTeX contents. You might want to read them as unicode instead. The `LatexToUnicodeProcessor` class solves this problem, but before adding the processor to the listener you must:

- [install Pandoc](http://pandoc.org/installing.html) in your system; and
- add [ryakad/pandoc-php](https://github.com/ryakad/pandoc-php) as a dependency of your project.

```php
use RenanBr\BibTexParser\Processor\LatexToUnicodeProcessor;

$listener->addProcessor(new LatexToUnicodeProcessor());
```

```bib
@article{
  title={Caf\\'{e}s and bars}
}
```

```
Array
(
    [0] => Array
        (
            [type] => article
            [title] => Cafés and bars
        )
)
```

Note: Order matters, add this processor as the last.

### Custom

The `Listener::addProcessor()` method expects a [callable] as argument. In the example shown below, we append the text `with laser` to the `title` tags for all entries.

```php
$listener->addProcessor(function (array $entry): array {
    $entry['title'] .= ' with laser';
    return $entry;
});
```

```
@article{
  title={BibTeX rocks}
}
```

```
Array
(
    [0] => Array
        (
            [type] => article
            [title] => BibTeX rocks with laser
        )
)
```

## Handling errors

This library throws two types of exception: `ParserException` and `ProcessorException`. The first one may happen during the data extraction. When it occurs it probably means the parsed BibTeX isn't valid. The second exception may be throwed during the data processing. When it occurs it means the listener's processors can't handle properly the data found. Both implement `ExceptionInterface`.

```php
use RenanBr\BibTexParser\Exception\ExceptionInterface;
use RenanBr\BibTexParser\Exception\ParserException;
use RenanBr\BibTexParser\Exception\ProcessorException;

try {
    // ... parser and listener configuration

    $parser->parseFile('/path/to/file.bib');
    $entries = $listener->export();
} catch (ParserException $exception) {
    // The BibTeX isn't valid
} catch (ProcessorException $exception) {
    // Listener's processors aren't able to handle data found
} catch (ExceptionInterface $exception) {
    // Alternatively, you can use this exception to catch all of them at once
}
```

## Advanced usage

The core of this library is constituted of these classes:

- `RenanBr\BibTexParser\Parser`: responsible for detecting units inside a BibTeX input;
- `RenanBr\BibTexParser\ListenerInterface`: responsible for treating units found.

You can attach listeners to the parser through `Parser::addListener()`. The parser is able to detect BibTeX units, such as "type", "tag's name", "tag's content". As the parser finds an unit, listeners are triggered.

You can code your own listener! All you have to do is handle units.

```php
interface RenanBr\BibTexParser\ListenerInterface
{
    /**
     * Called when an unit is found.
     *
     * @param string $text    The original content of the unit found.
     *                        Escape character will not be sent.
     * @param string $type    The type of unit found.
     *                        It can assume one of Parser's constant value.
     * @param array  $context Contains details of the unit found.
     */
    public function bibTexUnitFound(string $text, string $type, array $context): void;
}
```

`$type` may assume one of these values:

- `Parser::TYPE`
- `Parser::CITATION_KEY`
- `Parser::TAG_NAME`
- `Parser::RAW_TAG_CONTENT`
- `Parser::BRACED_TAG_CONTENT`
- `Parser::QUOTED_TAG_CONTENT`
- `Parser::ENTRY`

`$context` is an array with these keys:

- `offset` contains the `$text`'s beginning position.
  It may be useful, for example, to [seek on a file pointer](https://php.net/fseek);
- `length` contains the original `$text`'s length.
  It may differ from string length sent to the listener because may there are escaped characters.

[callable]: https://php.net/manual/en/language.types.callable.php
