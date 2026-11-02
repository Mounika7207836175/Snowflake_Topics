# XML File Format in Snowflake

## 1. What is XML?

XML stands for Extensible Markup Language. It represents structured information using tags and nested elements. It is text-based and hierarchical.

Example:

```xml
<employees>
    <employee>
        <id>101</id>
        <name>Ravi</name>
        <department>IT</department>
    </employee>
    <employee>
        <id>102</id>
        <name>Anu</name>
        <department>HR</department>
    </employee>
</employees>
```

## 2. XML concepts

- **Element:** a tag and its content, such as `<name>Ravi</name>`.
- **Attribute:** metadata inside an opening tag.
- **Parent element:** contains child elements.
- **Root element:** top-level element in the document.

Example with an attribute:

```xml
<employee id="101">
    <name>Ravi</name>
</employee>
```

## 3. XML compared with columnar formats

XML expresses hierarchy through nested tags. ORC and Parquet store typed data in a columnar binary layout.

XML is often used for exchanging structured documents between systems. ORC and Parquet are commonly used for analytical data storage.

## 4. XML in Snowflake

Create an XML file format:

```sql
CREATE FILE FORMAT XML_FORMAT
    TYPE = XML;
```

Read XML from a stage:

```sql
SELECT $1
FROM @MY_STAGE
(FILE_FORMAT => 'XML_FORMAT');
```

Snowflake provides XML parsing functions such as `PARSE_XML`:

```sql
SELECT PARSE_XML('<employee><id>101</id></employee>');
```

The parsed XML can be queried using Snowflake's semi-structured data capabilities. The exact path expressions depend on the parsed structure.

## 5. Practical workflow

1. Place the XML file in an internal or external stage.
2. Create an XML file format.
3. Confirm the file using `LIST`.
4. Read and inspect the XML structure.
5. Parse XML where required.
6. Decide whether to preserve the hierarchy or extract fields into relational columns.
7. Validate the result.

## 6. When is XML useful?

- Enterprise system integration.
- Hierarchical documents.
- Data exchange with applications that require XML.
- Systems that use XML-based APIs or message formats.

## 7. Key points

- XML is text-based and human-readable.
- It represents data with tags and nested elements.
- Attributes can store metadata.
- Snowflake file format type is `XML`.
- `PARSE_XML` can parse XML text into Snowflake's semi-structured representation.

## 8. Practice

1. Create an XML file format.
2. List an XML file in a stage.
3. Read the staged XML.
4. Try `PARSE_XML` with a small XML string.
5. Explore extracting an employee ID and name.
6. Compare the hierarchical XML structure with a relational table.
