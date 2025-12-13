<inputs>
  <input name="files" type="files" required>
</inputs>

<instructions>
  <instruction>
    Read and understand the Obsidian notebook practices defined in `.cursor/rules/base.mdc`, specifically:
    - The "Note Classification System" section - defines required frontmatter fields (status, priority, type) and their valid values
    - The "README Index Pattern" section - defines structure and guidelines for README index files
  </instruction>
  
  <instruction>
    For each file provided in {files}:
    <step>
      Read the file content to understand its current state and content
    </step>
    <step>
      Check if frontmatter already exists. If it does, preserve existing fields that are valid according to the classification system
    </step>
    <step>
      Analyze the document content to determine appropriate values:
      <analysis>
        - Assess completeness/progress level to determine `status` (seed, draft, developing, review, complete, archived)
        - Assess importance to determine `priority` (low, medium, high, critical)
        - Assess content category to determine `type` (note, pattern, strategy, reference, draft, decision)
        - Identify relevant topics for `tags` array based on content
        - If the file is named README.md, ensure it follows the README Index Pattern structure
      </analysis>
    </step>
    <step>
      Add or update frontmatter at the beginning of the file with:
      - `status`: appropriate value based on document completeness
      - `priority`: appropriate value based on document importance
      - `type`: appropriate value based on content category
      - `tags`: array of relevant topic tags (preserve existing tags if present and valid)
    </step>
    <step>
      Ensure frontmatter follows YAML format with proper indentation and is placed before any content
    </step>
    <step>
      If the document already has frontmatter, merge new classification fields with existing fields rather than replacing the entire frontmatter block
    </step>
  </instruction>
  
  <instruction>
    When determining classification values:
    <guidance>
      - Be conservative with `status: complete` - only use for polished, finished documents
      - Use `status: developing` for actively worked-on documents
      - Use `status: draft` for rough, incomplete notes
      - Use `status: seed` for fragments and quick captures
      - For `priority`, consider the document's role in decision-making and reference value
      - For `type`, match the primary purpose: patterns document reusable approaches, strategies document frameworks, references are guides/how-tos
      - Tags should be lowercase, descriptive, and help with cross-referencing
      - For README.md files: use `type: reference`, include `index` in tags, typically `priority: medium`, and ensure the file follows the README Index Pattern structure with Overview, Documentation Index, Quick Navigation, and Key Concepts sections
    </guidance>
  </instruction>
  
  <instruction>
    After updating each file, verify:
    <verification>
      - Frontmatter is valid YAML
      - All three required fields (status, priority, type) are present
      - Values match the allowed values from the classification system
      - Existing content is preserved unchanged
      - File structure remains intact
    </verification>
  </instruction>
</instructions>

