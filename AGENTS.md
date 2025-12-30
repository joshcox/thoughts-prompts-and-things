# AGENTS

<skills_system priority="1">

## Available Skills

<!-- SKILLS_TABLE_START -->
<usage>
When users ask you to perform tasks, check if any of the available skills below can help complete the task more effectively. Skills provide specialized capabilities and domain knowledge.

How to use skills:
- Invoke: Bash("openskills read <skill-name>")
- The skill content will load with detailed instructions on how to complete the task
- Base directory provided in output for resolving bundled resources (references/, scripts/, assets/)

Usage notes:
- Only use skills listed in <available_skills> below
- Do not invoke a skill that is already loaded in your context
- Each skill invocation is stateless
</usage>

<available_skills>

<skill>
<name>ai-structured-output</name>
<description>Generate type-safe AI content using Gemini structured output with Zod validation and Code Execution Tool. Use when building AI generation functions that need guaranteed output format.</description>
<location>project</location>
</skill>

<skill>
<name>hono-routing</name>
<description>|</description>
<location>project</location>
</skill>

<skill>
<name>mcp-builder</name>
<description>Guide for creating high-quality MCP (Model Context Protocol) servers that enable LLMs to interact with external services through well-designed tools. Use when building MCP servers to integrate external APIs or services, whether in Python (FastMCP) or Node/TypeScript (MCP SDK).</description>
<location>project</location>
</skill>

<skill>
<name>senior-frontend</name>
<description>Comprehensive frontend development skill for building modern, performant web applications using ReactJS, NextJS, TypeScript, Tailwind CSS. Includes component scaffolding, performance optimization, bundle analysis, and UI best practices. Use when developing frontend features, optimizing performance, implementing UI/UX designs, managing state, or reviewing frontend code.</description>
<location>project</location>
</skill>

<skill>
<name>shadcn-components</name>
<description>Add or customize shadcn/ui components in the shared UI package. Use when adding new components from shadcn registry or updating existing component variants.</description>
<location>project</location>
</skill>

</available_skills>
<!-- SKILLS_TABLE_END -->

</skills_system>
