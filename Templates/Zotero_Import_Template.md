# [{{title}}]({{desktopURI}})

 **Authors:** {{authors}}
 **Published:** {{date | format("YYYY")}}
 **Journal/Conference:** {{ publicationTitle or proceedingsTitle or bookTitle or university or institution }}
**Citekey:** `{{citekey}}`

---
## Abstract
> [!abstract]
>  ```
>  {{abstractNote}}
>  ```

---

{% persist "notes" %}
{%- if isFirstImport %}
## Summary
<!-- claude-summary-start -->
*Pending*
<!-- claude-summary-end -->
{%- endif %}
{% endpersist %}

## Notes & Highlights

{%- persist "obsidian-freeform-notes" %}
{%- if isFirstImport %}

### Freeform Notes

{%- endif %}
{%- endpersist %}

### From Zotero

{%- for annotation in annotations %}

---
{%- set callout = "quote" -%}
{%- set category = "Highlight" -%}

{%- if annotation.colorCategory == "Yellow" -%}
  {%- set callout = "note" -%}
  {%- set category = "Key Passage" -%}
{%- elif annotation.colorCategory == "Green" -%}
  {%- set callout = "success" -%}
  {%- set category = "Key Definition" -%}
{%- elif annotation.colorCategory == "Blue" -%}
  {%- set callout = "info" -%}
  {%- set category = "Methods -%}
{%- elif annotation.colorCategory == "Red" -%}
  {%- set callout = "danger" -%}
  {%- set category = "Critical Point" -%}
{%- elif annotation.colorCategory == "Gray" -%}
  {%- set callout = "abstract" -%}
  {%- set category = "Literature Review" -%}
{%- endif -%}

{%- if annotation.annotatedText %}
> [!{{callout}}] {{category}} (Page {{annotation.pageLabel}})
> {{annotation.annotatedText}}
{%- endif %}
{%- if annotation.imageRelativePath %}

![[{{annotation.imageRelativePath}}]]
{%- endif %}
{%- if annotation.comment %}
- **My Note:** {{annotation.comment}}
{%- endif %}
{%- persist annotation.id %}{%- endpersist %}
{%- endfor %}