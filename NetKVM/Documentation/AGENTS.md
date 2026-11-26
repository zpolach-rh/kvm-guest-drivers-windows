# NetKVM documentation rules

## Links from documentation to source code

VS Code's Markdown Preview opens a source link at the start of the file. Every link to a local source or header file in a NetKVM documentation page must therefore identify the exact function, method, type, variable, or macro to look for. Put the identifier immediately next to the link, or in the same table row in a clearly labeled symbol column. For example:

```markdown
`ParaNdis_FinishInitialization` in [ParaNdis_Common.cpp](../Common/ParaNdis_Common.cpp)
`NdisPollHandler::Register` in [ParaNdis_Poll.cpp](../wlh/ParaNdis_Poll.cpp)
```

Verify each identifier in the linked file before adding or changing the link. When a paragraph covers multiple steps in one file, name each relevant entry point or select one and explain why it is the starting point. Use qualified method names where needed to disambiguate. Do not use vague link labels such as "setup" or "implementation" without an adjacent identifier.

Keep links relative and avoid line-number anchors (`#L...`) and browser text fragments: they are unstable or do not work consistently in VS Code Markdown Preview. Do not claim that a file link jumps directly to a symbol. Links to diagrams, other documents, specifications, and whole project files can point to the file without a code symbol.
