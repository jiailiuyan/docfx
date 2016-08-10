# Validate Your Markdown Files

In Markdown, we can write any document with valid syntax. For example, Markdown supports to directly write HTML tag, we can write HTML tag `<h1>title</h1>` instead of Markdown syntax `#title`.
But for some purpose, some behaviors are unwanted, for example, you may not want to allow `<script>` tag in Markdown that can insert any javascript.

In this document, you'll learn how to define markdown validation rules, which will help you to validate markdown documents in an efficient way.

> Markdown validation is part of DFM, if you switch Markdown engine to other engine, validation might not work.

There're three kinds of validation rules provided by DocFX:

1. HTML tag rule, which is used to validate HTML tags in Markdown. There is a common need to restrict usage of HTML tags in Markdown to only allow "safe" HTML tags, so we created this built-in rule for you.
2. Markdown token rule. This can be used to validate different kinds of Markdown syntax elements, like headings, links, images, etc.
3. Metadata rule. This can be used to validate metadata of documents. Metadata can be defined in YAML header, `docfx.json`, or a single JSON file. Metadata rule gives you a central place to validate metadata against certain principle.

## HTML tag validation rules

For most cases, you may want to prohibit using certain html tags in markdown, so we built a built-in html tag rule for you.

To define a HTML tag rule, simply create a `md.style` with following content:

```json
{
   "tagRules": [
      {
         "tagNames": [ "H1", "H2" ],
         "behavior": "Warning",
         "messageFormatter": "Please do not use <H1> and <H2>, use '#' and '##' instead.",
         "customValidatorContractName": null,
         "openingTagOnly": false
      }
   ]
}
```

Then when anyone write `<H1>` or `<H2>` in Markdown file, it will give a warning.

You can use the following proprties to configure the HTML tag rule:

1.  `tagNames` is the list of HTML tag names to validate, *required*, *case-insensitive*.
2.  `behavior` defines the behavior when the HTML tag is met, *required*. Its value can be following:
    * None: Do nothing.
    * Warning: Log a warning.
    * Error: Log an error, it will break current build.
3.  `messageFormatter` is the log message when the HTML tag is hit, *required*.
    It can contain following variables:
    * `{0}` the name of tag.
    * `{1}` the whole tag.

    For example, the `messageFormatter` is `{0} is the tag name of {1}.`, and the tag is `<H1 class="heading">` match the rule, then it will output following message: `H1 is the tag name of <H1 class="heading">.`
4.  `customValidatorContractName` is an extension tag rule contract name for complex validation rule, *optional*.

    see [How to create a custom html tag validator](#how-to-create-a-custom-html-tag-validator).
5.  `openingTagOnly` is a boolean, *option*, default is `false`

    if `true`, it will only apply to opening tag, e.g. `<H1>`, otherwise, it will also apply to closing tag, e.g. `</H1>`.

### Test your rule

To enable your rule, put `md.style` in the same folder of `docfx.json`, then run `docfx`, warning will be shown if it encounters `<H1>` or `<H2>` during build.

### Create a custom HTML tag rule

By default HTML tag rule only validates whether an HTML tag exists in Markdown. Sometimes you may want to have additional validation against the content of the tag.
For example, you may not want a tag to contain `onclick` attribute as it can inject javascript to the page.
You can create a custom HTML tag rule to achieve this. 

1.  Create a project in your code editor (e.g. visual studio).
2.  Add nuget package `Microsoft.DocAsCode.Plugins` and `Microsoft.Composition`.
3.  Create a class and implement @Microsoft.DocAsCode.Plugins.ICustomMarkdownTagValidator.
4.  Add ExportAttribute with contract name.

For example, we require HTML link (`<a>`) should not contain `onclick` attribute:

```csharp
[Export("should_not_contain_onclick", typeof(ICustomMarkdownTagValidator))]
public class MyMarkdownTagValidator : ICustomMarkdownTagValidator
{
    public bool Validate(string tag)
    {
        // use Contains for demo purpose, a complete implementation should parse the HTML tag.
        return tag.Contains("onclick");
    }
}
```

And update your `md.style` with following content:

```json
{
   "tagRules": [
      {
         "tagNames": [ "a" ],
         "behavior": "Warning",
         "messageFormatter": "Please do not use 'onclick' in HTML link.",
         "customValidatorContractName": "should_not_contain_onclick",
         "openingTagOnly": true
      }
   ]
}
```

### How to enable custom HTML tag rules

1. Same as default HTML tag rule, config the rule in `md.style`.
2. Create a folder (`rules` for example) in your DocFX project folder, put all your custom rule assemblies to a `plugins` folder under `rules` folder.
   Now your DocFX project should look like this:

   ```
   /
   |- docfx.json
   |- md.style
   \- rules
      \- plugins
         \- <your_rule>.dll 
   ```
3. Update your `docfx.json` with following content:

   ```json
   {
     ...
     "dest": "_site",
     "template": [
      "default", "rules"
     ]
   }
   ```
4. Run `docfx` you'll see your rule being executed.

> The folder `rules` is actually a template folder. In DocFX, template is a place for you to customize build, render, validation behavior.
> For more information about template, please refer to our [template](howto_build_your_own_type_of_documentation_with_custom_plug-in.md) and [plugin](howto_build_your_own_type_of_documentation_with_custom_plug-in.md) documentation.

## Markdown token validation rules

Besides HTML tags, you may also want to validate Markdown syntax like heading or links. For example, in Markdown, you may want to limit code snippet to only support a set of languages.

To create such rule, follow the following steps:

1.  Create a project in your code editor (e.g. visual studio).
2.  Add nuget package `Microsoft.DocAsCode.MarkdownLite` and `Microsoft.Composition`.
3.  Create a class and implements @Microsoft.DocAsCode.MarkdownLite.IMarkdownTokenValidatorProvider
    > @Microsoft.DocAsCode.MarkdownLite.MarkdownTokenValidatorFactory contains some helper methods to create a validator.
4.  Add ExportAttribute with rule name.

For example, the following rule require all code block to be `csharp`:
```csharp
[Export("code_snippet_should_be_csharp", typeof(IMarkdownTokenValidatorProvider))]
public class MyMarkdownTokenValidatorProvider : IMarkdownTokenValidatorProvider
{
    public ImmutableArray<IMarkdownTokenValidator> GetValidators()
    {
        return ImmutableArray.Create(
            MarkdownTokenValidatorFactory.FromLambda<MarkdownCodeBlockToken>(t =>
            {
                if (t.Lang != "csharp")
                {
                     throw new DocumentException($"Code lang {t.Lang} is not valid, in file: {t.SourceInfo.File}, at line: {t.SourceInfo.LineNumber}");
                }
            }));
    }
}
```

To enable this rule, update your `md.style` to the following:

```json
{
    "rules": [ "code_snippet_should_be_csharp" ]
}
```

Then follow the same steps in [How to enable custom HTML tag rules](#how-to-enable-custom-html-tag-rules), run `docfx` you'll see your rule executed.

### Logging in your rules

As you can see in the above example, you can throw @Microsoft.DocAsCode.Plugins.DocumentException to raise an error, this will stop the build immediately.

You can also use @Microsoft.DocAsCode.Common.Logger.LogWarning(System.String,System.String,System.String,System.String) and @Microsoft.DocAsCode.Common.Logger.LogError(System.String,System.String,System.String,System.String) to report a warning and an error respectively.

> To use these methods, you need to install nuget package `Microsoft.DocAsCode.Common` first.

The different between `ReportError` and throw `DocumentException` is throw exception will stop the build immediately but `ReportError` won't stop build but will eventually fail the build after rules are run.

## Advanced usage of `md.style`

### Default rules

If a rule has the contract name of `default`, it will be enabled by default. You don't need to enable it in `md.style`.

### Enable/disable rules in `md.style`

You can add use `disable` to specify whether disable a rule:
```json
{
   "rules": [ { "name": "<rule_name>", "disable": true } ]
}
```

This gives you an opportunity to disable the rules enabled by default.

## Validate metadata in markdown files

In markdown file, we can write some metadata in [conceptual](../spec/docfx_flavored_markdown.md#yaml-header) or [overwrite document](intro_overwrite_files.md).
And we allow add some plug-ins to validate metadata written in markdown files.

### Scope of metadata validation

Metadata is coming multiple sources, the following metadata will be validated during build: 
1.  YAML header in markdown.
2.  Global metadata and file metaata in `docfx.json`.
3.  Global metadata and file metadata defined in separate `.json` files.

> For more information about global metadata and global metadata, see [docfx.json format](docfx.exe_user_manual.md#3-docfx-json-format).

### Create validation plug-ins

1.  Create a project in your code editor (e.g. visual studio).
2.  Add nuget package `Microsoft.DocAsCode.Plugins` and `Microsoft.Composition`.
3.  Create a class and implement @Microsoft.DocAsCode.Plugins.IInputMetadataValidator

For example, the following validator prohibits any metadata with name `hello`:

```csharp
[Export(typeof(IInputMetadataValidator))]
public class MyInputMetadataValidator : IInputMetadataValidator
{
    public void Validate(string sourceFile, ImmutableDictionary<string, object> metadata)
    {
        if (metadata.ContainsKey("hello"))
        {
            throw new DocumentException($"Metadata 'hello' is not allowed, file: {sourceFile}");
        }
    }
}
```

Enable metadata rule is same as other rules, just copy the assemblies to the `plugins` of your template folder and run `docfx`.