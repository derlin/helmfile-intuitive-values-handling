# Example: a simple way to handle values in helmfile

[helmfile](https://github.com/helmfile/helmfile) is a very powerful tool, but his way of handling release values is daunting for the beginners (and experts).
This repo proposes a simple pattern to handle values, that is completely generic and works in all situations.

## TL;DR

This repo reproduces the way values are handled in umbrella charts (Helm Charts with sub-charts).

**If you don't like to read** but want to experiment instead:
- clone this repo,
- customize the different release values by editing `environments/default.yaml`,
- override values per environment by editing `environments/<envName>.yaml` (available environements are `local` and `prod`),
- see for yourself how your changes work by running: `helmfile -e <env> write-values` and see the output.

To apply this in your helmfile:

1. ensure you reference `env-magic.gotmpl` under *all* release values (`releases.<releaseName>.values`) and,
2. ensure you reference both `default.yaml` and `<envName>.yaml` files under `environments.<envName>.values`.

Done.

--------------

<!-- TOC start -->
- [🐌 Before we start: what are values](#-before-we-start-what-are-values)
- [👎 How helmfile handles values](#-how-helmfile-handles-values)
  * [Values and global values](#values-and-global-values)
  * [Value overrides per environment](#value-overrides-per-environment)
- [👌 Handling values better in helmfile](#-handling-values-better-in-helmfile)
  * [What it entails](#what-it-entails)
  * [How it works](#how-it-works)
<!-- TOC end -->

## 🐌 Before we start: what are values

Values are parameters passed to a Helm chart. In helmfile, all releases are instances of a given helm chart, that defines its parameters in a `values.yaml` file. Those parameters can be overridden using different mechanisms (`--set` and `--values` using `helm`).

We often distinguish between two kind of values:
1. *release-specific values*: values that need to be set only for one release 
   (we will refer to then as simply *values* in this README), and
2. *global values*: values that must be set in all charts, such as the `domain` or the URL to some kafka broker.
   Global values are especially useful when most of the releases in a helmfile rely on a common library chart.

Furthermore, some values may need to be set differently depending on the environment.
We can thus talk about *global values* and *global environment values*.

## 👎 How helmfile handles values

### Values and global values

In a helmfile, *values* can be overriden at the release level:
```yaml
releases:
  - name: foo
    # ...
    values:
      - simple: value # raw value 
      - path/to/raw-yaml-file.yaml # values read as is from a file
      - path/to/template-outputting-yaml.gotmpl # go template outputting valid YAML values
```

In order to set *global values*, one has to reference the same values/file/template below each releases, as seen above. There is no built-in support per se.

### Value overrides per environment

When we need to set *environment values*, helmfile supports a `environments.<envName>.values` property:
```yaml
releases: [...]
environments:
   prod:
     values:
      - global: value # raw value 
      - path/to/raw-yaml-file.yaml # values read as is from a file
      - path/to/template-outputting-yaml.gotmpl # go template outputting valid YAML values
```

However, unintuitively enough, this **doesn't set any value at all**, but simply makes them available to templates.
That is, in go templates attached to releases, the `.Values` will be populated with the environment values (on top of the other values defined at the release level), such that a template may do:
```yaml
global: {{ .Values | get "global" "UNSET" }}
```
Notice the convoluted syntax, which is necessary because `global` may be undefined in other environments.

When all environment values are meant to be global, one trick is to pass the following template inline: `{{ toYaml  .Values | nindent 10 }}`.

Take this helmfile for example:
```yaml
releases:
  - name: foo
    # ... reference some chart here
    values:
      - release-specific: check
      - {{ toYaml .Values | nindent 10 }} # <-- output all values (env + normal)
environments:
   prod:
     values:
       - env-prod: check
```
The output of `helmfile -e prod write-values` will be:
```yaml
release-specific: check
env-prod: check
```

This is nice for global environment values, but what about release-specific ones ?
Well, there is no way around writing a specific template for each release... Or is there ?

## 👌 Handling values better in helmfile

In this repository is shown a simpler way of handling values in helmfile, that doesn't require release-specific templates and covers all value types (global/specific, per environment).

Basically, it reproduces the way values are handled in umbrella charts (Helm Charts with subcharts).

### What it entails

The idea is:

* define release-specific values in `environments/default.yaml`, under the release name;
* define global values in `environments/default.yaml`, under `global`;
* for environment values, use the same logic, but in a different file (e.g. `environments/<envName>.yaml`).

For this pattern to work:

1. all environments must reference `environments/default.yaml`, plus their specific file. This is also necessary for the default environment ! However, in this case, only the default file is listed.
2. all releases must reference `env-magic.gotmpl` in their `values`. To simplify and keep it DRY, you can use YAML anchors (see the example `helmfile.yaml`).

### How it works

All the magic comes from the `env-magic.gotmpl` file, which consists of one single line:
```go
{{ merge (.Values | get .Release.Name  dict) (.Values | get "global" dict) | toYaml }}
```

Since helmfile merges environment values and normal values automatically and makes them available through `.Values` in templates, the only thing left to do is to select only the values for the current release (`.Release.Name`),
and the global values (`global`). We then merge the two, giving precedence to the release-specific values, 
and render them as YAML.