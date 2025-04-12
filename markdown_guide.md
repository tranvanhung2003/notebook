# Table of Contents <!-- omit from toc -->

- [1. Markdown Guide](#1-markdown-guide)
  - [1.1. Basic Syntax](#11-basic-syntax)
    - [1.1.1. Headings](#111-headings)
    - [1.1.2. Paragraphs](#112-paragraphs)
    - [1.1.3. Line Breaks](#113-line-breaks)
    - [1.1.4. Emphasis](#114-emphasis)
      - [1.1.4.1. Bold](#1141-bold)
      - [1.1.4.2. Italic](#1142-italic)
      - [1.1.4.3. Bold and Italic](#1143-bold-and-italic)
    - [1.1.5. Blockquotes](#115-blockquotes)
      - [1.1.5.1. Blockquotes with Multiple Paragraphs](#1151-blockquotes-with-multiple-paragraphs)
      - [1.1.5.2. Nested Blockquotes](#1152-nested-blockquotes)
      - [1.1.5.3. Blockquotes with Other Elements](#1153-blockquotes-with-other-elements)
    - [1.1.6. Lists](#116-lists)
      - [1.1.6.1. Ordered Lists](#1161-ordered-lists)
      - [1.1.6.2. Unordered Lists](#1162-unordered-lists)
      - [1.1.6.3. Adding Elements in Lists](#1163-adding-elements-in-lists)
        - [1.1.6.3.1. Paragraphs](#11631-paragraphs)
        - [1.1.6.3.2. Blockquotes](#11632-blockquotes)
        - [1.1.6.3.3. Code Blocks](#11633-code-blocks)
        - [1.1.6.3.4. Images](#11634-images)
        - [1.1.6.3.5. Lists](#11635-lists)
    - [1.1.7. Code](#117-code)
      - [1.1.7.1. Escaping Backticks](#1171-escaping-backticks)
      - [1.1.7.2. Code Blocks](#1172-code-blocks)
    - [1.1.8. Horizontal Rules](#118-horizontal-rules)
    - [1.1.9. Links](#119-links)
      - [1.1.9.1. Adding Titles](#1191-adding-titles)
      - [1.1.9.2. URLs and Email Addresses](#1192-urls-and-email-addresses)
      - [1.1.9.3. Formatting Links](#1193-formatting-links)
      - [1.1.9.4. Reference-style Links](#1194-reference-style-links)
        - [1.1.9.4.1. Formatting the First Part of the Link](#11941-formatting-the-first-part-of-the-link)
        - [1.1.9.4.2. Formatting the Second Part of the Link](#11942-formatting-the-second-part-of-the-link)
        - [1.1.9.4.3. An Example Putting the Parts Together](#11943-an-example-putting-the-parts-together)
    - [1.1.10. Images](#1110-images)
      - [1.1.10.1. Linking Images](#11101-linking-images)
    - [1.1.11. Escaping Characters](#1111-escaping-characters)
    - [1.1.12. HTML](#1112-html)
  - [1.2. Extended Syntax](#12-extended-syntax)
    - [1.2.1. Tables](#121-tables)
      - [1.2.1.1. Alignment](#1211-alignment)
    - [1.2.2. Fenced Code Blocks](#122-fenced-code-blocks)
      - [1.2.2.1. Syntax Highlighting](#1221-syntax-highlighting)
    - [1.2.3. Heading IDs](#123-heading-ids)
      - [1.2.3.1. Linking to Heading IDs](#1231-linking-to-heading-ids)
    - [1.2.4. Strikethrough](#124-strikethrough)
    - [1.2.5. Task Lists](#125-task-lists)
    - [1.2.6. Automatic URL Linking](#126-automatic-url-linking)
    - [1.2.7. Disabling Automatic URL Linking](#127-disabling-automatic-url-linking)
  - [1.3. Hacks](#13-hacks)
    - [1.3.1. Indent (Tab)](#131-indent-tab)
    - [1.3.2. Comments](#132-comments)
    - [1.3.3. Table Formatting](#133-table-formatting)
      - [1.3.3.1. Line Breaks Within Table Cells](#1331-line-breaks-within-table-cells)
    - [1.3.4. Table of Contents](#134-table-of-contents)
    - [1.3.5. Videos](#135-videos)

# 1. [Markdown Guide](https://www.markdownguide.org/)

## 1.1. [Basic Syntax](https://www.markdownguide.org/basic-syntax/)

### 1.1.1. Headings

```
# Heading level 1

## Heading level 2

### Heading level 3

#### Heading level 4

##### Heading level 5

###### Heading level 6
```

### 1.1.2. Paragraphs

```
I really like using Markdown.

I think I'll use it to format all of my documents from now on.
```

### 1.1.3. Line Breaks

```
First line with the HTML tag after.<br>
And the next line.
```

### 1.1.4. Emphasis

#### 1.1.4.1. Bold

```
I just love **bold text**.
```

#### 1.1.4.2. Italic

```
Italicized text is the *cat's meow*.

Italicized text is the _cat's meow_.
```

#### 1.1.4.3. Bold and Italic

```
This text is ***really important***.

This text is **_really important_**.
```

### 1.1.5. Blockquotes

```
> Dorothy followed her through many of the beautiful rooms in her castle.
```

#### 1.1.5.1. Blockquotes with Multiple Paragraphs

```
> Dorothy followed her through many of the beautiful rooms in her castle.
>
> The Witch bade her clean the pots and kettles and sweep the floor and keep the fire fed with wood.
```

#### 1.1.5.2. Nested Blockquotes

```
> Dorothy followed her through many of the beautiful rooms in her castle.
>
>> The Witch bade her clean the pots and kettles and sweep the floor and keep the fire fed with wood.
```

#### 1.1.5.3. Blockquotes with Other Elements

```
> #### The quarterly results look great!
>
> - Revenue was off the chart.
> - Profits were higher than ever.
>
>  *Everything* is going according to **plan**.
```

### 1.1.6. Lists

#### 1.1.6.1. Ordered Lists

```
1. First item
2. Second item
3. Third item
    1. Indented item
    2. Indented item
4. Fourth item
```

#### 1.1.6.2. Unordered Lists

```
- First item
- Second item
- Third item
    - Indented item
    - Indented item
- Fourth item
```

#### 1.1.6.3. Adding Elements in Lists

##### 1.1.6.3.1. Paragraphs

```
* This is the first list item.
* Here's the second list item.

    I need to add another paragraph below the second list item.

* And here's the third list item.
```

##### 1.1.6.3.2. Blockquotes

```
* This is the first list item.
* Here's the second list item.

    > A blockquote would look great below the second list item.

* And here's the third list item.
```

##### 1.1.6.3.3. Code Blocks

```
1. Open the file.
2. Find the following code block on line 21:

        <html>
          <head>
            <title>Test</title>
          </head>

3. Update the title to match the name of your website.
```

##### 1.1.6.3.4. Images

```
1. Open the file containing the Linux mascot.
2. Marvel at its beauty.

    ![Tux, the Linux mascot](/assets/images/tux.png)

3. Close the file.
```

##### 1.1.6.3.5. Lists

```
1. First item
2. Second item
3. Third item
    - Indented item
    - Indented item
4. Fourth item
```

### 1.1.7. Code

#### 1.1.7.1. Escaping Backticks

```
At the command prompt, type `nano`.

``Use `code` in your Markdown file.``
```

#### 1.1.7.2. Code Blocks

```
    <html>
      <head>
      </head>
    </html>
```

### 1.1.8. Horizontal Rules

```
---
```

### 1.1.9. Links

```
My favorite search engine is [Duck Duck Go](https://duckduckgo.com).
```

#### 1.1.9.1. Adding Titles

```
My favorite search engine is [Duck Duck Go](https://duckduckgo.com "The best search engine for privacy").
```

#### 1.1.9.2. URLs and Email Addresses

```
<https://www.markdownguide.org>
<fake@example.com>
```

#### 1.1.9.3. Formatting Links

```
I love supporting the **[EFF](https://eff.org)**.
This is the *[Markdown Guide](https://www.markdownguide.org)*.
See the section on [`code`](#code).
```

#### 1.1.9.4. Reference-style Links

##### 1.1.9.4.1. Formatting the First Part of the Link

```
[hobbit-hole][1]
```

##### 1.1.9.4.2. Formatting the Second Part of the Link

```
[1]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle
[1]: https://en.wikipedia.org/wiki/Hobbit#Lifestyle "Hobbit lifestyles"
```

##### 1.1.9.4.3. An Example Putting the Parts Together

```
In a hole in the ground there lived a hobbit. Not a nasty, dirty, wet hole, filled with the ends of worms and an oozy smell, nor yet a dry, bare, sandy hole with nothing in it to sit down on or to eat: it was a [hobbit-hole][1], and that means comfort.

[1]: <https://en.wikipedia.org/wiki/Hobbit#Lifestyle> "Hobbit lifestyles"
```

### 1.1.10. Images

```
![The San Juan Mountains are beautiful!](/assets/images/san-juan-mountains.jpg "San Juan Mountains")
```

#### 1.1.10.1. Linking Images

```
[![An old rock in the desert](/assets/images/shiprock.jpg "Shiprock, New Mexico by Beau Rogers")](https://www.flickr.com/photos/beaurogers/31833779864/in/photolist-Qv3rFw-34mt9F-a9Cmfy-5Ha3Zi-9msKdv-o3hgjr-hWpUte-4WMsJ1-KUQ8N-deshUb-vssBD-6CQci6-8AFCiD-zsJWT-nNfsgB-dPDwZJ-bn9JGn-5HtSXY-6CUhAL-a4UTXB-ugPum-KUPSo-fBLNm-6CUmpy-4WMsc9-8a7D3T-83KJev-6CQ2bK-nNusHJ-a78rQH-nw3NvT-7aq2qf-8wwBso-3nNceh-ugSKP-4mh4kh-bbeeqH-a7biME-q3PtTf-brFpgb-cg38zw-bXMZc-nJPELD-f58Lmo-bXMYG-bz8AAi-bxNtNT-bXMYi-bXMY6-bXMYv)
```

### 1.1.11. Escaping Characters

```
\* Without the backslash, this would be a bullet in an unordered list.
```

| Character | Name                |
| --------- | ------------------- |
| `\`       | backslash           |
| `` ` ``   | backtick            |
| `*`       | asterisk            |
| `_`       | underscore          |
| `{ }`     | curly braces        |
| `[ ]`     | brackets            |
| `< >`     | angle brackets      |
| `( )`     | parentheses         |
| `#`       | pound sign          |
| `+`       | plus sign           |
| `-`       | minus sign (hyphen) |
| `.`       | dot                 |
| `!`       | exclamation mark    |
| `\|`      | pipe                |

### 1.1.12. HTML

```
This **word** is bold. This <em>word</em> is italic.
```

## 1.2. [Extended Syntax](https://www.markdownguide.org/extended-syntax/)

### 1.2.1. Tables

```
| Syntax      | Description |
| ----------- | ----------- |
| Header      | Title       |
| Paragraph   | Text        |
```

#### 1.2.1.1. Alignment

```
| Syntax      | Description | Test Text     |
| :---        |    :----:   |          ---: |
| Header      | Title       | Here's this   |
| Paragraph   | Text        | And more      |
```

### 1.2.2. Fenced Code Blocks

```
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

#### 1.2.2.1. Syntax Highlighting

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "age": 25
}
```

### 1.2.3. Heading IDs

```
### My Great Heading {#custom-id}
```

#### 1.2.3.1. Linking to Heading IDs

```
[My Great Heading](#custom-id)
```

### 1.2.4. Strikethrough

```
~~The world is flat.~~ We now know that the world is round.
```

### 1.2.5. Task Lists

```
- [x] Write the press release
- [ ] Update the website
- [ ] Contact the media
```

### 1.2.6. Automatic URL Linking

http://www.example.com

### 1.2.7. Disabling Automatic URL Linking

`http://www.example.com`

## 1.3. [Hacks](https://www.markdownguide.org/hacks/)

### 1.3.1. Indent (Tab)

&nbsp;&nbsp;&nbsp;&nbsp;This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph. This is the first sentence of my indented paragraph.

### 1.3.2. Comments

```
Here's a paragraph that will be visible.

[This is a comment that will be hidden.]: #

And here's another paragraph that's visible.
```

```
[comment]: <> "This is a comment"
[comment]: <> (This is a comment)

[//]: <> "This is also a comment."
[//]: <> (This is also a comment.)

[//]: # "This is also a comment."
[//]: # (This is also a comment.)
```

### 1.3.3. Table Formatting

#### 1.3.3.1. Line Breaks Within Table Cells

```
| Syntax    | Description                                 |
| --------- | ------------------------------------------- |
| Header    | Title                                       |
| Paragraph | First paragraph. <br><br> Second paragraph. |
```

### 1.3.4. Table of Contents

```
#### Table of Contents

- [Underline](#underline)
- [Indent](#indent)
- [Center](#center)
- [Color](#color)
```

### 1.3.5. Videos

```
[![Image alt text](https://img.youtube.com/vi/YOUTUBE-ID/0.jpg)](https://www.youtube.com/watch?v=YOUTUBE-ID)
```

[![Less Than Jake — Scott Farcas Takes It On The Chin](https://img.youtube.com/vi/PYCxct2e0zI/0.jpg)](https://www.youtube.com/watch?v=PYCxct2e0zI)
