# Common Guide
If there is any memory or data type, it should be clearly shown with its structure (size and data type).
And, for each data, add enough explanation to understand clearly.

# Common Guide for pipeline diagram
Draw mermaid diagram with keeping the styles described below.
Provide top diagram showing module, interfance, and pipeline clearly.
  : You can provide each modules diagram also (Pipeline should be clearly shown.)
Define working scenarios and explain each with detailed sequence diagram


1. Pipeline should be clearly shown including pipeline registers.
2. Each logic should be clearly shown as pseudo code upon relevant request.
3. The action via each connection should be clearly noted.
4. To generate mermaid code safely, use the synatx like "   s3 --> o3["S3 resp + s3_redirect"]" when noting text.

# Draw diagram with mermaid for pipeline, sequence and data structure also
1. Include possible explanation 
2. If possible, keep the styling.
3. The shape of memory block (SRAM) should be cylinder
4. Use "<br>" instead of "\n" since "/n" does not work.
```
  %% styling
  classDef memory fill:#e8f5e9,stroke:#1b5e20,stroke-width:1px;
  classDef reg fill:#fff3e0,stroke:#e65100,stroke-width:1px;
  classDef logic fill:#e1f5fe,stroke:#01579b,stroke-width:1px;
  classDef io fill:#fce4ec,stroke:#880e4f,stroke-width:1px;
```

