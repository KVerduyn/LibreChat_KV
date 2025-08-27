# LibreChat Artifacts for Traffic Data Visualization

This document outlines the recommended approach for incorporating interactive visualizations into the GADER Traffic MCP Server using LibreChat's Artifacts system.

## Overview

LibreChat's **Artifacts** system allows AI assistants to create interactive, executable code components that render directly in the chat interface. This provides an ideal solution for displaying traffic measurement data through maps, charts, and interactive dashboards without requiring any modifications to LibreChat itself.

## How Artifacts Work

### Technology Stack
- **Sandpack** (CodeSandbox framework) for code execution
- **React/TypeScript** environment for interactive components  
- **Tailwind CSS + shadcn/ui** for styling
- **Pre-installed libraries** including Recharts, Radix UI, Three.js, Mermaid

### The Rendering Process
1. **AI generates** artifact syntax in chat response
2. **LibreChat parses** the artifact tags
3. **Sandpack environment** creates isolated execution context
4. **Component renders** in dedicated panel alongside chat
5. **User interacts** with live, functional component

## Supported Artifact Types

```typescript
'application/vnd.react'     // Interactive React components
'application/vnd.mermaid'   // Mermaid diagrams  
'text/html'                 // Static HTML pages
'application/vnd.code-html' // HTML with code highlighting
```

## Pre-installed Libraries

Your artifacts automatically have access to:

**UI Components:**
- All **Radix UI** components (dialogs, buttons, forms, etc.)
- **Tailwind CSS** for styling
- **Lucide React** icons

**Visualization:**
- **Recharts** for charts/graphs ✨
- **Three.js** for 3D graphics
- **Mermaid** for diagrams ✨

**Utilities:**
- **date-fns** for date handling
- **class-variance-authority** for conditional styling

## Implementation Strategy for GADER Traffic MCP

### Phase 1: Enhanced MCP Tools

Update existing MCP tools to return artifact-ready data:

#### 1. Enhanced `discover_locations` Tool
```javascript
// Return format that includes artifact suggestion
{
  "type": "location-list",
  "locations": [
    {
      "id": "http://example.org/location/1", 
      "lat": 51.2194, 
      "lng": 4.4025, 
      "name": "Oostende Center",
      "description": "Traffic measurement point in city center"
    }
  ],
  "total-count": 15,
  "artifact_data": {
    "center": [51.2194, 4.4025],
    "zoom": 13,
    "city": "Oostende"
  }
}
```

#### 2. New `format_results_as_map` Tool
```javascript
// Input: location data from discover_locations
// Output: React component code for interactive map
{
  "type": "artifact",
  "format": "application/vnd.react",
  "title": "Traffic Measurement Locations in Oostende",
  "content": "/* React component code for interactive map */"
}
```

#### 3. New `format_results_as_chart` Tool
```javascript
// Input: traffic measurement data
// Output: Recharts component for data visualization
{
  "type": "artifact", 
  "format": "application/vnd.react",
  "title": "Traffic Data Visualization",
  "content": "/* Recharts component code */"
}
```

### Phase 2: Artifact Templates

#### Interactive Map Template
```tsx
import { useState } from 'react';

// Note: For production, would need react-leaflet added to dependencies
// Currently using placeholder map implementation
function TrafficMap({ locations, center, zoom }) {
  const [selectedLocation, setSelectedLocation] = useState(null);
  
  return (
    <div className="h-96 w-full bg-gray-100 rounded-lg border relative">
      <div className="absolute inset-0 flex items-center justify-center">
        <div className="text-center">
          <h3 className="text-lg font-semibold mb-2">Traffic Measurement Locations</h3>
          <div className="space-y-2">
            {locations.map(location => (
              <button
                key={location.id}
                onClick={() => setSelectedLocation(location)}
                className="block w-full text-left p-2 bg-white rounded border hover:bg-blue-50"
              >
                <div className="font-medium">{location.name}</div>
                <div className="text-sm text-gray-600">{location.description}</div>
                <div className="text-xs text-gray-400">
                  Coordinates: {location.lat}, {location.lng}
                </div>
              </button>
            ))}
          </div>
        </div>
      </div>
      
      {selectedLocation && (
        <div className="absolute top-4 right-4 bg-white p-3 rounded shadow">
          <h4 className="font-semibold">{selectedLocation.name}</h4>
          <p className="text-sm">{selectedLocation.description}</p>
        </div>
      )}
    </div>
  );
}
```

#### Traffic Data Chart Template
```tsx
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import { useState } from 'react';

function TrafficChart({ data, title }) {
  const [selectedMetric, setSelectedMetric] = useState('cars');
  
  const metrics = [
    { key: 'cars', label: 'Cars', color: '#8884d8' },
    { key: 'bikes', label: 'Bicycles', color: '#82ca9d' },
    { key: 'pedestrians', label: 'Pedestrians', color: '#ffc658' },
    { key: 'trucks', label: 'Trucks', color: '#ff7300' }
  ];
  
  return (
    <div className="w-full h-96 p-4 bg-white rounded-lg border">
      <h3 className="text-lg font-semibold mb-4">{title}</h3>
      
      <div className="flex gap-2 mb-4">
        {metrics.map(metric => (
          <button
            key={metric.key}
            onClick={() => setSelectedMetric(metric.key)}
            className={`px-3 py-1 rounded text-sm ${
              selectedMetric === metric.key 
                ? 'bg-blue-500 text-white' 
                : 'bg-gray-200 hover:bg-gray-300'
            }`}
          >
            {metric.label}
          </button>
        ))}
      </div>
      
      <ResponsiveContainer width="100%" height={300}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis dataKey="time" />
          <YAxis />
          <Tooltip />
          <Legend />
          {metrics
            .filter(m => selectedMetric === 'all' || m.key === selectedMetric)
            .map(metric => (
              <Line 
                key={metric.key}
                type="monotone" 
                dataKey={metric.key} 
                stroke={metric.color}
                strokeWidth={2}
              />
            ))}
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
}
```

### Phase 3: Enhanced User Workflow

```
1. User: "Show traffic locations in Oostende"
   ↓
2. Agent calls: discover_locations({location: "Oostende"})
   ↓
3. MCP returns: location data + artifact template
   ↓
4. Agent creates: Interactive map artifact showing all locations
   ↓
5. User: Clicks on specific location in map
   ↓
6. Agent calls: translate_nl_to_sparql + execute_sparql_query
   ↓
7. Agent creates: Chart artifact showing traffic data for that location
```

## Example Artifact Syntax

When the AI assistant wants to create an artifact, it uses this syntax:

```markdown
I'll create an interactive map showing all traffic measurement locations in Oostende:

<artifacts>
<artifact identifier="traffic-map-oostende" type="application/vnd.react" title="Traffic Measurement Locations in Oostende">
import { useState } from 'react';

function TrafficMap() {
  const locations = [
    { id: 1, lat: 51.2194, lng: 4.4025, name: "Oostende Center", count: 1234 },
    { id: 2, lat: 51.2150, lng: 4.3980, name: "Harbor District", count: 987 }
  ];
  
  const [selectedLocation, setSelectedLocation] = useState(null);
  
  return (
    <div className="h-96 w-full bg-gradient-to-br from-blue-50 to-blue-100 rounded-lg border-2 border-blue-200 relative overflow-hidden">
      {/* Map visualization */}
      <div className="absolute inset-4">
        <h3 className="text-xl font-bold text-blue-800 mb-4 text-center">
          Traffic Measurement Locations - Oostende
        </h3>
        
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4 h-full">
          {locations.map(location => (
            <div
              key={location.id}
              onClick={() => setSelectedLocation(location)}
              className="bg-white rounded-lg p-4 shadow-md hover:shadow-lg transition-shadow cursor-pointer border-l-4 border-blue-500"
            >
              <div className="flex justify-between items-start">
                <div>
                  <h4 className="font-semibold text-gray-800">{location.name}</h4>
                  <p className="text-sm text-gray-600">
                    Coordinates: {location.lat}°N, {location.lng}°E
                  </p>
                  <p className="text-xs text-gray-500 mt-2">
                    Total measurements: {location.count.toLocaleString()}
                  </p>
                </div>
                <div className="w-3 h-3 bg-red-500 rounded-full animate-pulse"></div>
              </div>
            </div>
          ))}
        </div>
        
        {selectedLocation && (
          <div className="absolute top-0 right-0 bg-white p-4 rounded-bl-lg shadow-lg border-l-4 border-green-500">
            <h5 className="font-semibold text-green-800">Selected Location</h5>
            <p className="text-sm">{selectedLocation.name}</p>
            <button 
              onClick={() => setSelectedLocation(null)}
              className="text-xs text-gray-500 hover:text-gray-700 mt-2"
            >
              Click to deselect
            </button>
          </div>
        )}
      </div>
    </div>
  );
}

export default TrafficMap;
</artifact>
</artifacts>
```

## Technical Implementation Steps

### 1. Update MCP Server Tools

Modify existing tools in `/gader/logic/mcp-server/src/clojure/gader/logic/mcp_server/tools.clj`:

```clojure
;; Enhanced discover_locations that includes artifact data
(defn discover-locations
  "Discover traffic measurement locations and prepare for artifact rendering"
  [{:keys [location transport-modes date-range]}]
  (let [query-result (execute-sparql-query {:query sparql-query})]
    (if (:error query-result)
      query-result
      {:type "location-list"
       :locations (map format-location-for-artifact bindings)
       :total-count (count bindings)
       :artifact-data {:center [lat lng] :zoom 13 :city location}})))
```

### 2. Add New MCP Tools

```clojure
;; New tool for generating map artifacts
(defn generate-map-artifact
  "Generate interactive map artifact from location data"
  [{:keys [locations city]}]
  {:type "artifact-suggestion"
   :format "application/vnd.react"
   :title (format "Traffic Measurement Locations in %s" city)
   :template "traffic-map"
   :data {:locations locations :city city}})

;; New tool for generating chart artifacts  
(defn generate-chart-artifact
  "Generate traffic data chart artifact"
  [{:keys [data title chart-type]}]
  {:type "artifact-suggestion"
   :format "application/vnd.react"
   :title title
   :template "traffic-chart"
   :data {:data data :chart-type (or chart-type "line")}})
```

### 3. Update Protocol Definition

Add new tools to the MCP protocol in `protocol.clj`:

```clojure
{:name "generate_map_artifact"
 :description "Generate interactive map artifact for traffic measurement locations"
 :inputSchema {:type "object"
               :properties {:locations {:type "array"}
                          :city {:type "string"}}}}

{:name "generate_chart_artifact"
 :description "Generate interactive chart artifact for traffic data visualization"
 :inputSchema {:type "object"
               :properties {:data {:type "array"}
                          :title {:type "string"}
                          :chart-type {:type "string"}}}}
```

## Benefits of This Approach

✅ **Zero LibreChat modifications** - leverages existing artifact system  
✅ **Professional interactive UI** - not just static images  
✅ **Rich component ecosystem** - charts, forms, interactive elements  
✅ **Responsive design** - works on mobile and desktop  
✅ **Real-time updates** - artifacts can be regenerated with new data  
✅ **Familiar development** - standard React/TypeScript patterns  
✅ **Security** - sandboxed execution environment  

## Limitations & Considerations

❌ **Limited to pre-installed libraries** - cannot add external NPM packages  
❌ **No persistent state** - artifacts reset when regenerated  
❌ **No direct API calls** - all data must flow through MCP tools  
❌ **Sandboxed environment** - some browser APIs may be restricted  

## Future Enhancements

1. **Map Integration**: Add react-leaflet to LibreChat's standard dependencies
2. **Real-time Updates**: Implement WebSocket connections for live data
3. **Export Functionality**: Add data export capabilities to artifacts
4. **Advanced Filtering**: Interactive filters for time ranges, vehicle types
5. **Collaborative Features**: Share artifacts between users

## Conclusion

LibreChat's Artifacts system provides the ideal foundation for rich, interactive traffic data visualizations. By leveraging the existing infrastructure and following React/TypeScript patterns, we can create professional, user-friendly interfaces for exploring traffic measurement data without requiring any modifications to LibreChat itself.

The approach scales from simple location maps to complex interactive dashboards, making it perfect for both end-users and analysts working with traffic data from the GADER system.