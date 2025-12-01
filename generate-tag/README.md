# generate tag action

A template with parameters is passed as input. A rendered string is returned as output.

## Examples

```yaml
# nosemgrep
- uses: identw/build-images-action/generate-tag@main
  id: tag
  with:
    template: "{{ area }}-{{ dateTime }}-{{ ref }}-{{ commit }}"
    params: |
      area: ${{ inputs.area }}

- run: |
    echo '${{ steps.tag.outputs.result }}'
```

There are standard template strings that are automatically replaced and do not need to be specified in the parameters. For example, `dateTime`. All of them are described below in the `params` parameter description.

## Inputs

### `template`
**Required** 
```
default: none
```
String template

### `params`
**Required** 
```
default: none
```
Parameters passed to the template.

#### Standard templates
1. `{{ dateTime }}` - automatically replaced with the current UTC time in `YYYYMMDDhhmm` format
1. `{{ ref }}` - replaced with `github.ref_name`
1. `{{ commit }}` - replaced with `github.sha`, but only the short version (first 10 characters)

## Outputs

### `result`
Result of the rendered string
