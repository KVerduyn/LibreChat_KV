# MCP Server Workflow Diagram

This document describes the complete workflow of the GADER Traffic MCP Server, including all tools, backend services, and data flows.

## Mermaid Diagram

```mermaid
graph TB
    %% External Systems
    LC[LibreChat] 
    TS[Translator Service<br/>100.64.85.30:5000]
    VS[Virtuoso Database<br/>100.124.101.98:8890]
    FS[SPARQL Formatter<br/>gader-sparql-formatter:8080]

    %% MCP Server Components
    subgraph MCP["MCP Server (localhost:8090)"]
        HTTP[HTTP Handler<br/>gader.logic.mcp-server.http]
        PROTO[Protocol Handler<br/>gader.logic.mcp-server.protocol]
        TOOLS[Tools Handler<br/>gader.logic.mcp-server.tools]
        SESS[Session Manager<br/>gader.logic.mcp-server.session]
    end

    %% Tools
    subgraph TOOLBOX["Available MCP Tools"]
        T1[get_system_prompt]
        T2[discover_locations]
        T3[select_locations] 
        T4[translate_nl_to_sparql]
        T5[execute_sparql_query]
        T6[format_sparql_results]
    end

    %% Workflow Steps
    LC -->|1. HTTP POST /mcp<br/>JSON-RPC request| HTTP
    HTTP --> PROTO
    PROTO --> TOOLS

    %% Tool 1: System Prompt
    TOOLS -->|get_system_prompt| T1
    T1 -->|Read instructions file<br/>from container| T1

    %% Tool 2: Discover Locations (FIRST STEP)
    TOOLS -->|discover_locations<br/>{location: "Oostende"}| T2
    T2 -->|Format: "Toon alle meetlocaties in {location}"| T4
    T4 -->|HTTP POST /messages<br/>{question, thread_id}| TS
    TS -->|SPARQL query for locations| T4
    T4 --> T5
    T5 -->|Execute SPARQL| VS
    VS -->|Raw location results| T5
    T5 -->|Location URIs + WKT| T2
    T2 -->|{type:"location-list", locations:[{id, wkt}]}| LC

    %% Tool 3: Select Locations (SECOND STEP)  
    TOOLS -->|select_locations<br/>{location-ids: ["uri1","uri2"]}| T3
    T3 -->|Store in session| SESS
    SESS -->|selected-location-ids: [...]| SESS

    %% Tool 4: Translate Query (THIRD STEP)
    TOOLS -->|translate_nl_to_sparql<br/>{question: "traffic query"}| T4
    T4 -->|Get selected locations| SESS
    T4 -->|HTTP POST /messages<br/>{question, selectedLocationIds, thread_id}| TS
    TS -->|SPARQL with location filters| T4

    %% Tool 5: Execute Query (FOURTH STEP)
    TOOLS -->|execute_sparql_query<br/>{query: "SPARQL..."}| T5
    T5 -->|HTTP POST /sparql<br/>Content-Type: application/sparql-query| VS
    VS -->|application/sparql-results+json| T5

    %% Tool 6: Format Results (OPTIONAL)
    TOOLS -->|format_sparql_results<br/>{sparql_results, query_type}| T6
    T6 -->|HTTP POST /format-results<br/>{sparql_results, query_type, query}| FS
    FS -->|Formatted table/map results| T6

    %% Response Flow
    T1 --> TOOLS
    T2 --> TOOLS  
    T3 --> TOOLS
    T4 --> TOOLS
    T5 --> TOOLS
    T6 --> TOOLS
    TOOLS --> PROTO
    PROTO -->|JSON-RPC response| HTTP
    HTTP -->|HTTP response| LC

    %% Session State
    subgraph STATE["Session State"]
        TID[thread-id:<br/>mcp-session-uuid]
        LOCS[selected-location-ids:<br/>["uri1", "uri2"]]
    end
    SESS <--> STATE

    %% Styling
    classDef external fill:#e1f5fe
    classDef mcp fill:#f3e5f5  
    classDef tools fill:#e8f5e8
    classDef state fill:#fff3e0
    
    class LC,TS,VS,FS external
    class HTTP,PROTO,TOOLS,SESS mcp
    class T1,T2,T3,T4,T5,T6 tools
    class STATE,TID,LOCS state
```

## Workflow Overview

### **System Architecture**

The MCP Server acts as a bridge between LibreChat and various backend services:

- **LibreChat**: The main chat interface that sends user queries
- **MCP Server** (`localhost:8090`): Orchestrates tool calls and manages session state
- **Translator Service** (`100.64.85.30:5000`): Converts Dutch questions to SPARQL queries
- **Virtuoso Database** (`100.124.101.98:8890`): Stores traffic measurement data in RDF format
- **SPARQL Formatter** (`gader-sparql-formatter:8080`): Formats raw SPARQL results for display

### **Available Tools**

1. **`get_system_prompt`**
   - **Purpose**: Returns Dutch domain instructions for traffic measurements
   - **Input**: None
   - **Output**: Domain-specific instructions text
   - **Backend**: Reads from local container file

2. **`discover_locations`** ⭐ **(ALWAYS FIRST STEP)**
   - **Purpose**: Find available measurement locations in a geographic area
   - **Input**: `{location: "city name"}`
   - **Output**: `{type: "location-list", locations: [{id: "uri", wkt: "coordinates"}], total-count: N}`
   - **Backend Flow**: 
     - Calls `translate_nl_to_sparql` internally
     - Sends request to Translator Service
     - Executes SPARQL against Virtuoso Database
     - Returns location URIs and coordinates

3. **`select_locations`** ⭐ **(SECOND STEP)**
   - **Purpose**: Choose specific locations for subsequent data queries
   - **Input**: `{location-ids: ["uri1", "uri2", ...]}`
   - **Output**: `{message: "Selected N locations", selected-count: N}`
   - **Backend**: Stores location IDs in session state

4. **`translate_nl_to_sparql`** ⭐ **(THIRD STEP)**
   - **Purpose**: Convert Dutch traffic questions to SPARQL queries
   - **Input**: `{question: "Dutch traffic question"}`
   - **Output**: `{type: "sparql-query", query: "SPARQL..."}`
   - **Backend**: 
     - Retrieves selected locations from session
     - Sends to Translator Service with location context
     - Returns SPARQL query with location filters

5. **`execute_sparql_query`** ⭐ **(FOURTH STEP)**
   - **Purpose**: Execute SPARQL query against traffic database
   - **Input**: `{query: "SPARQL query string"}`
   - **Output**: `{results: {...}, query: "...", row-count: N}`
   - **Backend**: Direct HTTP POST to Virtuoso SPARQL endpoint

6. **`format_sparql_results`** *(OPTIONAL)*
   - **Purpose**: Format raw SPARQL results for better display
   - **Input**: `{sparql_results: {...}, query_type: "auto|table|map"}`
   - **Output**: Structured table or map format
   - **Backend**: Calls SPARQL Formatter service

### **Correct Usage Sequence**

For any traffic measurement query, the agent should follow this sequence:

```
1. discover_locations → Find measurement points in the area
2. select_locations   → Choose specific locations to query  
3. translate_nl_to_sparql → Convert question to SPARQL with location context
4. execute_sparql_query → Run the query against database
5. format_sparql_results → (Optional) Format for display
```

### **Session Management**

Each conversation maintains state:

- **Thread ID**: Unique identifier for the conversation (`mcp-session-{uuid}`)
- **Selected Locations**: Array of measurement location URIs
- **Persistence**: Session state persists across multiple tool calls

### **Key Features**

- **Location-First Approach**: All traffic queries must first identify measurement locations
- **Session Context**: Selected locations are remembered within a conversation
- **Multi-Service Integration**: Coordinates between translator, database, and formatter services
- **Error Handling**: Comprehensive error responses for tool failures
- **Flexible Formatting**: Optional result formatting for different display needs

### **Example Complete Workflow**

**User Question**: "Hoeveel auto's zijn geteld op 1 augustus 2024 in Oostende?"

```
1. discover_locations({location: "Oostende"})
   → Returns: {locations: [{id: "uri1", wkt: "POINT(...)"}, ...]}

2. select_locations({location-ids: ["uri1", "uri2"]})
   → Stores: selected-location-ids = ["uri1", "uri2"]

3. translate_nl_to_sparql({question: "hoeveel auto's zijn geteld op 1 augustus 2024"})
   → Sends: {question: "...", selectedLocationIds: ["uri1", "uri2"], thread_id: "..."}
   → Returns: SPARQL query with location filters

4. execute_sparql_query({query: "SELECT COUNT(...) WHERE {...}"})
   → Returns: {results: {...}, row-count: 1}

5. format_sparql_results({sparql_results: {...}, query_type: "table"})
   → Returns: Formatted table with car count
```

This architecture ensures that all traffic measurement queries are properly scoped to specific geographic locations before executing data queries.