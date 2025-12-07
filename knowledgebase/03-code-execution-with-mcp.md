# Code Execution with MCP: Building More Efficient Agents

Source: https://www.anthropic.com/engineering/code-execution-with-mcp

## Overview

This article explores how code execution environments can enhance AI agents' efficiency when working with the Model Context Protocol (MCP). The core insight is that agents perform better when writing code to call tools rather than using direct tool calls.

## Key Problems Solved

### Token Consumption Issues
1. Tool definitions consume excessive context when loaded upfront
2. Intermediate results must pass through the model multiple times

**Example:** When downloading a meeting transcript and attaching it to a Salesforce record, the full transcript flows through the model twice—once on retrieval, again on insertion.

## The Solution: Code APIs Instead of Direct Calls

Rather than exposing tools as direct function calls, present MCP servers as TypeScript/code APIs. Agents then write code to interact with these tools.

### File Structure Example
```
servers/
├── google-drive/
│   ├── getDocument.ts
│   └── index.ts
├── salesforce/
│   ├── updateRecord.ts
│   └── index.ts
```

### Agent Code Example
```typescript
const transcript = (await gdrive.getDocument({documentId: 'abc123'})).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: transcript }
});
```

**Impact:** Reduces token usage from 150,000 to 2,000 tokens—a 98.7% savings.

## Key Benefits

### Progressive Disclosure
Models navigate filesystems efficiently, loading only needed tool definitions on demand rather than loading all upfront.

### Context-Efficient Results
Agents filter and transform large datasets in the execution environment. For 10,000-row spreadsheets, show only filtered results (e.g., 5 rows) instead of all rows.

### Powerful Control Flow
Loops, conditionals, and error handling execute as code rather than requiring repeated model calls. This improves latency and reduces token consumption.

### Privacy-Preserving Operations
Intermediate results stay in the execution environment. Sensitive data can be tokenized automatically—agents see placeholders like `[EMAIL_1]` while real data flows between tools without entering the model context.

### State Persistence
Agents save intermediate results to files, enabling resumable workflows and building a reusable skill library over time.

## Important Considerations

Code execution introduces infrastructure complexity:
- Secure sandboxing required
- Resource limits essential
- Monitoring needed

These operational requirements must be weighed against token savings and performance gains.

## Key Takeaways

- Present MCP servers as code APIs instead of direct tool calls
- Intermediate results stay in execution environment, not model context
- Progressive disclosure loads only needed tool definitions
- Code execution enables loops, conditionals, and error handling without repeated model calls
- Privacy is enhanced by keeping sensitive data out of model context
- Infrastructure complexity is the tradeoff for efficiency gains
