# 기본 수기아마 레이아웃 기능적용

I'll create a simple JavaScript implementation of the Sugiyama algorithm for hierarchical graph layout.

```html:graph_layout.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Graph Layout with Sugiyama Algorithm</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
        }
        canvas {
            border: 1px solid #ccc;
            margin-top: 20px;
        }
        button {
            margin: 10px 5px;
            padding: 8px 15px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <h1>Graph Layout with Sugiyama Algorithm</h1>
    <div>
        <button id="generateGraph">Generate Random Graph</button>
        <button id="layoutGraph">Apply Layout</button>
    </div>
    <canvas id="graphCanvas" width="800" height="600"></canvas>

    <script>
        class Graph {
            constructor() {
                this.nodes = [];
                this.edges = [];
            }

            addNode(id, label) {
                this.nodes.push({
                    id,
                    label,
                    x: 0,
                    y: 0,
                    layer: 0,
                    order: 0
                });
                return this;
            }

            addEdge(source, target) {
                this.edges.push({
                    source,
                    target
                });
                return this;
            }

            getNode(id) {
                return this.nodes.find(node => node.id === id);
            }

            getNodeIndex(id) {
                return this.nodes.findIndex(node => node.id === id);
            }

            // Get all outgoing edges from a node
            getOutgoingEdges(nodeId) {
                return this.edges.filter(edge => edge.source === nodeId);
            }

            // Get all incoming edges to a node
            getIncomingEdges(nodeId) {
                return this.edges.filter(edge => edge.target === nodeId);
            }
        }

        class SugiyamaLayout {
            constructor(graph) {
                this.graph = graph;
                this.layers = [];
            }

            // Step 1: Assign nodes to layers
            assignLayers() {
                // Reset layers
                this.graph.nodes.forEach(node => { node.layer = 0; });
                
                let assigned = new Set();
                let queue = [];
                
                // Find root nodes (nodes with no incoming edges)
                this.graph.nodes.forEach(node => {
                    if (this.graph.getIncomingEdges(node.id).length === 0) {
                        queue.push(node.id);
                        assigned.add(node.id);
                        node.layer = 0;
                    }
                });
                
                // BFS to assign layers
                while (queue.length > 0) {
                    const currentId = queue.shift();
                    const currentNode = this.graph.getNode(currentId);
                    
                    this.graph.getOutgoingEdges(currentId).forEach(edge => {
                        const targetNode = this.graph.getNode(edge.target);
                        targetNode.layer = Math.max(targetNode.layer, currentNode.layer + 1);
                        
                        if (!assigned.has(edge.target)) {
                            queue.push(edge.target);
                            assigned.add(edge.target);
                        }
                    });
                }
                
                // Organize nodes into layer arrays
                this.layers = [];
                let maxLayer = 0;
                
                this.graph.nodes.forEach(node => {
                    maxLayer = Math.max(maxLayer, node.layer);
                });
                
                for (let i = 0; i <= maxLayer; i++) {
                    this.layers[i] = this.graph.nodes.filter(node => node.layer === i);
                }
                
                return this;
            }
            
            // Step 2: Order nodes within layers to minimize crossings
            minimizeCrossings() {
                for (let i = 1; i < this.layers.length; i++) {
                    this.orderLayer(i);
                }
                return this;
            }
            
            orderLayer(layerIndex) {
                if (layerIndex <= 0 || layerIndex >= this.layers.length) return;
                
                const currentLayer = this.layers[layerIndex];
                const prevLayer = this.layers[layerIndex - 1];
                
                // Simple ordering: give each node a barycenter value based on connected nodes in previous layer
                currentLayer.forEach(node => {
                    const incomingEdges = this.graph.getIncomingEdges(node.id);
                    if (incomingEdges.length === 0) {
                        node.barycenter = 0;
                        return;
                    }
                    
                    let sum = 0;
                    incomingEdges.forEach(edge => {
                        const sourceNode = this.graph.getNode(edge.source);
                        const sourceIndex = prevLayer.findIndex(n => n.id === sourceNode.id);
                        if (sourceIndex >= 0) {
                            sum += sourceIndex;
                        }
                    });
                    
                    node.barycenter = sum / incomingEdges.length;
                });
                
                // Sort the layer by barycenter values
                currentLayer.sort((a, b) => a.barycenter - b.barycenter);
                
                // Update order in layer
                currentLayer.forEach((node, index) => {
                    node.order = index;
                });
            }
            
            // Step 3: Assign x, y coordinates
            assignCoordinates() {
                const nodeWidth = 80;
                const nodeHeight = 40;
                const layerHeight = 120;
                
                for (let i = 0; i < this.layers.length; i++) {
                    const layer = this.layers[i];
                    const layerWidth = layer.length * nodeWidth;
                    const startX = (800 - layerWidth) / 2;
                    
                    layer.forEach((node, index) => {
                        node.x = startX + index * nodeWidth + nodeWidth / 2;
                        node.y = i * layerHeight + 60;
                    });
                }
                
                return this;
            }
            
            // Run the complete layout algorithm
            run() {
                this.assignLayers()
                    .minimizeCrossings()
                    .assignCoordinates();
                return this.graph;
            }
        }

        // Render the graph
        function renderGraph(graph, canvas) {
            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw edges
            ctx.strokeStyle = '#999';
            ctx.lineWidth = 1;
            
            graph.edges.forEach(edge => {
                const sourceNode = graph.getNode(edge.source);
                const targetNode = graph.getNode(edge.target);
                
                ctx.beginPath();
                ctx.moveTo(sourceNode.x, sourceNode.y);
                ctx.lineTo(targetNode.x, targetNode.y);
                ctx.stroke();
                
                // Draw arrow
                const angle = Math.atan2(targetNode.y - sourceNode.y, targetNode.x - sourceNode.x);
                const arrowSize = 10;
                ctx.beginPath();
                ctx.moveTo(targetNode.x - arrowSize * Math.cos(angle) + arrowSize * Math.sin(angle) / 2,
                           targetNode.y - arrowSize * Math.sin(angle) - arrowSize * Math.cos(angle) / 2);
                ctx.lineTo(targetNode.x, targetNode.y);
                ctx.lineTo(targetNode.x - arrowSize * Math.cos(angle) - arrowSize * Math.sin(angle) / 2,
                           targetNode.y - arrowSize * Math.sin(angle) + arrowSize * Math.cos(angle) / 2);
                ctx.fillStyle = '#999';
                ctx.fill();
            });
            
            // Draw nodes
            graph.nodes.forEach(node => {
                ctx.fillStyle = '#3498db';
                ctx.beginPath();
                ctx.arc(node.x, node.y, 20, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.fillStyle = 'white';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.font = '12px Arial';
                ctx.fillText(node.label, node.x, node.y);
            });
        }

        // Generate random graph
        function generateRandomGraph(nodeCount = 10, edgeDensity = 0.3) {
            const graph = new Graph();
            
            // Add nodes
            for (let i = 0; i < nodeCount; i++) {
                graph.addNode(i, `Node ${i}`);
            }
            
            // Add edges (ensuring it's a DAG for Sugiyama to work properly)
            for (let i = 0; i < nodeCount; i++) {
                for (let j = i + 1; j < nodeCount; j++) {
                    if (Math.random() < edgeDensity) {
                        graph.addEdge(i, j);
                    }
                }
            }
            
            return graph;
        }

        // Main code
        document.addEventListener('DOMContentLoaded', () => {
            const canvas = document.getElementById('graphCanvas');
            let graph = null;
            
            document.getElementById('generateGraph').addEventListener('click', () => {
                graph = generateRandomGraph();
                
                // Assign initial random positions
                graph.nodes.forEach(node => {
                    node.x = Math.random() * 700 + 50;
                    node.y = Math.random() * 500 + 50;
                });
                
                renderGraph(graph, canvas);
            });
            
            document.getElementById('layoutGraph').addEventListener('click', () => {
                if (!graph) {
                    alert('Please generate a graph first!');
                    return;
                }
                
                const layout = new SugiyamaLayout(graph);
                graph = layout.run();
                renderGraph(graph, canvas);
            });
        });
    </script>
</body>
</html>
```
완성했습니다. 이 프로그램은 Sugiyama 알고리즘을 이용해 방향성 비순환 그래프(DAG)를 계층적으로 배치하는 자바스크립트 구현체입니다.

주요 기능:
- 랜덤 그래프 생성
- Sugiyama 알고리즘을 통한 그래프 레이아웃 정렬
- Canvas를 통한 그래프 시각화

Sugiyama 알고리즘의 기본 단계:
1. 노드를 계층으로 할당 (assignLayers)
2. 각 계층 내 교차 최소화 (minimizeCrossings)
3. 노드에 실제 좌표 할당 (assignCoordinates)

파일을 열면 "Generate Random Graph" 버튼으로 랜덤 그래프를 생성하고, "Apply Layout" 버튼으로 Sugiyama 알고리즘을 적용해 그래프를 정렬할 수 있습니다.


# 여기에 제약조건을 줄거야 어떠한 노드들은 그룹화 될거고 그룹화된 노드는 서로 묶여있기 때문에 가로 범위를 벗어날 수 없다는 (같은 가로상의 범위 내에서는 수기아마 알고리즘이 그대로 적용되나 다른 그룹으로 ) 조건 하에 레이아웃이 되어야 해.


그룹화된 노드들이 같은 가로 범위 내에 위치하도록 코드를 수정하겠습니다.

```html:graph_layout.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Graph Layout with Sugiyama Algorithm and Node Grouping</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
        }
        canvas {
            border: 1px solid #ccc;
            margin-top: 20px;
        }
        button {
            margin: 10px 5px;
            padding: 8px 15px;
            cursor: pointer;
        }
        .controls {
            margin-bottom: 15px;
        }
        .group-area {
            margin-top: 15px;
            padding: 10px;
            border: 1px solid #ddd;
            background-color: #f9f9f9;
            max-width: 800px;
        }
        .group-item {
            margin: 5px 0;
            padding: 5px;
            border: 1px solid #eee;
            background-color: #fff;
        }
    </style>
</head>
<body>
    <h1>Graph Layout with Sugiyama Algorithm and Node Grouping</h1>
    <div class="controls">
        <button id="generateGraph">Generate Random Graph</button>
        <button id="addGroup">Add Group</button>
        <button id="layoutGraph">Apply Layout</button>
    </div>
    
    <div id="groupsContainer" class="group-area">
        <h3>Node Groups</h3>
        <div id="groupsList"></div>
    </div>
    
    <canvas id="graphCanvas" width="800" height="600"></canvas>

    <script>
        class Graph {
            constructor() {
                this.nodes = [];
                this.edges = [];
                this.groups = []; // Added: store groups of nodes
            }

            addNode(id, label) {
                this.nodes.push({
                    id,
                    label,
                    x: 0,
                    y: 0,
                    layer: 0,
                    order: 0,
                    group: null // Default: node belongs to no group
                });
                return this;
            }

            addEdge(source, target) {
                this.edges.push({
                    source,
                    target
                });
                return this;
            }

            getNode(id) {
                return this.nodes.find(node => node.id === id);
            }

            getNodeIndex(id) {
                return this.nodes.findIndex(node => node.id === id);
            }

            // Get all outgoing edges from a node
            getOutgoingEdges(nodeId) {
                return this.edges.filter(edge => edge.source === nodeId);
            }

            // Get all incoming edges to a node
            getIncomingEdges(nodeId) {
                return this.edges.filter(edge => edge.target === nodeId);
            }
            
            // Create a new group and add nodes to it
            createGroup(groupId, nodeIds) {
                // Mark nodes as belonging to this group
                nodeIds.forEach(nodeId => {
                    const node = this.getNode(nodeId);
                    if (node) {
                        node.group = groupId;
                    }
                });
                
                // Add group info
                this.groups.push({
                    id: groupId,
                    nodes: nodeIds,
                    minX: 0,
                    maxX: 0
                });
                
                return this;
            }
            
            // Get nodes in a specific group
            getNodesInGroup(groupId) {
                return this.nodes.filter(node => node.group === groupId);
            }
            
            // Get group by ID
            getGroup(groupId) {
                return this.groups.find(group => group.id === groupId);
            }
        }

        class SugiyamaLayout {
            constructor(graph) {
                this.graph = graph;
                this.layers = [];
            }

            // Step 1: Assign nodes to layers
            assignLayers() {
                // Reset layers
                this.graph.nodes.forEach(node => { node.layer = 0; });
                
                let assigned = new Set();
                let queue = [];
                
                // Find root nodes (nodes with no incoming edges)
                this.graph.nodes.forEach(node => {
                    if (this.graph.getIncomingEdges(node.id).length === 0) {
                        queue.push(node.id);
                        assigned.add(node.id);
                        node.layer = 0;
                    }
                });
                
                // BFS to assign layers
                while (queue.length > 0) {
                    const currentId = queue.shift();
                    const currentNode = this.graph.getNode(currentId);
                    
                    this.graph.getOutgoingEdges(currentId).forEach(edge => {
                        const targetNode = this.graph.getNode(edge.target);
                        targetNode.layer = Math.max(targetNode.layer, currentNode.layer + 1);
                        
                        if (!assigned.has(edge.target)) {
                            queue.push(edge.target);
                            assigned.add(edge.target);
                        }
                    });
                }
                
                // Organize nodes into layer arrays
                this.layers = [];
                let maxLayer = 0;
                
                this.graph.nodes.forEach(node => {
                    maxLayer = Math.max(maxLayer, node.layer);
                });
                
                for (let i = 0; i <= maxLayer; i++) {
                    this.layers[i] = this.graph.nodes.filter(node => node.layer === i);
                }
                
                return this;
            }
            
            // Step 2: Order nodes within layers to minimize crossings
            minimizeCrossings() {
                for (let i = 1; i < this.layers.length; i++) {
                    this.orderLayer(i);
                }
                
                // Additional step: ensure grouped nodes are adjacent within each layer
                this.ensureGroupsAdjacentInLayers();
                
                return this;
            }
            
            // Ensure nodes in the same group are adjacent in their respective layers
            ensureGroupsAdjacentInLayers() {
                // For each layer
                for (let layerIndex = 0; layerIndex < this.layers.length; layerIndex++) {
                    const layer = this.layers[layerIndex];
                    
                    // Group nodes by their group
                    const groupedNodes = {};
                    layer.forEach(node => {
                        if (node.group !== null) {
                            if (!groupedNodes[node.group]) {
                                groupedNodes[node.group] = [];
                            }
                            groupedNodes[node.group].push(node);
                        }
                    });
                    
                    // Reorder nodes in the layer to keep grouped nodes adjacent
                    if (Object.keys(groupedNodes).length > 0) {
                        // First, assign positions to ungrouped nodes
                        let currentPos = 0;
                        const newOrder = [];
                        
                        // Add ungrouped nodes
                        layer.filter(node => node.group === null).forEach(node => {
                            node.order = currentPos++;
                            newOrder.push(node);
                        });
                        
                        // Add grouped nodes (ensuring they're adjacent)
                        for (const groupId in groupedNodes) {
                            const groupNodes = groupedNodes[groupId];
                            groupNodes.sort((a, b) => a.barycenter - b.barycenter);
                            
                            // Assign consecutive orders
                            groupNodes.forEach(node => {
                                node.order = currentPos++;
                                newOrder.push(node);
                            });
                        }
                        
                        // Replace the layer with the new ordering
                        this.layers[layerIndex] = newOrder;
                    }
                }
            }
            
            orderLayer(layerIndex) {
                if (layerIndex <= 0 || layerIndex >= this.layers.length) return;
                
                const currentLayer = this.layers[layerIndex];
                const prevLayer = this.layers[layerIndex - 1];
                
                // Simple ordering: give each node a barycenter value based on connected nodes in previous layer
                currentLayer.forEach(node => {
                    const incomingEdges = this.graph.getIncomingEdges(node.id);
                    if (incomingEdges.length === 0) {
                        node.barycenter = 0;
                        return;
                    }
                    
                    let sum = 0;
                    incomingEdges.forEach(edge => {
                        const sourceNode = this.graph.getNode(edge.source);
                        const sourceIndex = prevLayer.findIndex(n => n.id === sourceNode.id);
                        if (sourceIndex >= 0) {
                            sum += sourceIndex;
                        }
                    });
                    
                    node.barycenter = sum / incomingEdges.length;
                });
                
                // Sort the layer by barycenter values
                currentLayer.sort((a, b) => a.barycenter - b.barycenter);
                
                // Update order in layer
                currentLayer.forEach((node, index) => {
                    node.order = index;
                });
            }
            
            // Step 3: Assign x, y coordinates
            assignCoordinates() {
                const nodeWidth = 80;
                const nodeHeight = 40;
                const layerHeight = 120;
                
                // First assign standard coordinates
                for (let i = 0; i < this.layers.length; i++) {
                    const layer = this.layers[i];
                    const layerWidth = layer.length * nodeWidth;
                    const startX = (800 - layerWidth) / 2;
                    
                    layer.forEach((node, index) => {
                        node.x = startX + index * nodeWidth + nodeWidth / 2;
                        node.y = i * layerHeight + 60;
                    });
                }
                
                // Calculate group boundaries
                this.calculateGroupBoundaries(nodeWidth);
                
                // Adjust node positions to satisfy group constraints
                this.adjustNodePositionsForGroups(nodeWidth);
                
                return this;
            }
            
            // Calculate min and max X values for each group across all layers
            calculateGroupBoundaries(nodeWidth) {
                // Reset group boundaries
                this.graph.groups.forEach(group => {
                    group.minX = Infinity;
                    group.maxX = -Infinity;
                });
                
                // Calculate initial group boundaries
                this.graph.nodes.forEach(node => {
                    if (node.group !== null) {
                        const group = this.graph.getGroup(node.group);
                        if (group) {
                            group.minX = Math.min(group.minX, node.x - nodeWidth/2);
                            group.maxX = Math.max(group.maxX, node.x + nodeWidth/2);
                        }
                    }
                });
            }
            
            // Adjust node positions to keep grouped nodes in the same horizontal range
            adjustNodePositionsForGroups(nodeWidth) {
                for (let i = 0; i < this.layers.length; i++) {
                    const layer = this.layers[i];
                    
                    // Group nodes by their group in this layer
                    const groupedNodesInLayer = {};
                    layer.forEach(node => {
                        if (node.group !== null) {
                            if (!groupedNodesInLayer[node.group]) {
                                groupedNodesInLayer[node.group] = [];
                            }
                            groupedNodesInLayer[node.group].push(node);
                        }
                    });
                    
                    // For each group in this layer, adjust positions
                    for (const groupId in groupedNodesInLayer) {
                        const group = this.graph.getGroup(groupId);
                        const groupNodes = groupedNodesInLayer[groupId];
                        
                        if (group && groupNodes.length > 0) {
                            // Calculate group center
                            const groupWidth = group.maxX - group.minX;
                            const groupCenter = group.minX + groupWidth / 2;
                            
                            // Calculate current center of nodes in this layer
                            let currentCenter = 0;
                            groupNodes.forEach(node => {
                                currentCenter += node.x;
                            });
                            currentCenter /= groupNodes.length;
                            
                            // Calculate adjustment needed
                            const adjustment = groupCenter - currentCenter;
                            
                            // Apply adjustment to all nodes in this group in this layer
                            groupNodes.forEach(node => {
                                node.x += adjustment;
                            });
                        }
                    }
                }
            }
            
            // Run the complete layout algorithm
            run() {
                this.assignLayers()
                    .minimizeCrossings()
                    .assignCoordinates();
                return this.graph;
            }
        }

        // Render the graph
        function renderGraph(graph, canvas) {
            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw group backgrounds
            renderGroups(graph, ctx);
            
            // Draw edges
            ctx.strokeStyle = '#999';
            ctx.lineWidth = 1;
            
            graph.edges.forEach(edge => {
                const sourceNode = graph.getNode(edge.source);
                const targetNode = graph.getNode(edge.target);
                
                ctx.beginPath();
                ctx.moveTo(sourceNode.x, sourceNode.y);
                ctx.lineTo(targetNode.x, targetNode.y);
                ctx.stroke();
                
                // Draw arrow
                const angle = Math.atan2(targetNode.y - sourceNode.y, targetNode.x - sourceNode.x);
                const arrowSize = 10;
                ctx.beginPath();
                ctx.moveTo(targetNode.x - arrowSize * Math.cos(angle) + arrowSize * Math.sin(angle) / 2,
                           targetNode.y - arrowSize * Math.sin(angle) - arrowSize * Math.cos(angle) / 2);
                ctx.lineTo(targetNode.x, targetNode.y);
                ctx.lineTo(targetNode.x - arrowSize * Math.cos(angle) - arrowSize * Math.sin(angle) / 2,
                           targetNode.y - arrowSize * Math.sin(angle) + arrowSize * Math.cos(angle) / 2);
                ctx.fillStyle = '#999';
                ctx.fill();
            });
            
            // Draw nodes
            graph.nodes.forEach(node => {
                // Node color: different if grouped
                ctx.fillStyle = node.group !== null ? getGroupColor(node.group) : '#3498db';
                ctx.beginPath();
                ctx.arc(node.x, node.y, 20, 0, Math.PI * 2);
                ctx.fill();
                
                ctx.fillStyle = 'white';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.font = '12px Arial';
                ctx.fillText(node.label, node.x, node.y);
            });
        }
        
        // Render group backgrounds
        function renderGroups(graph, ctx) {
            graph.groups.forEach((group, index) => {
                // Find min/max x and y for the group
                let minX = Infinity, minY = Infinity;
                let maxX = -Infinity, maxY = -Infinity;
                
                graph.getNodesInGroup(group.id).forEach(node => {
                    minX = Math.min(minX, node.x - 30);
                    maxX = Math.max(maxX, node.x + 30);
                    minY = Math.min(minY, node.y - 30);
                    maxY = Math.max(maxY, node.y + 30);
                });
                
                if (minX < Infinity) {
                    // Draw group background
                    ctx.fillStyle = getGroupColor(group.id, 0.2);
                    ctx.strokeStyle = getGroupColor(group.id);
                    ctx.lineWidth = 2;
                    
                    const padding = 10;
                    ctx.beginPath();
                    ctx.roundRect(
                        minX - padding, 
                        minY - padding, 
                        maxX - minX + padding * 2, 
                        maxY - minY + padding * 2,
                        10
                    );
                    ctx.fill();
                    ctx.stroke();
                    
                    // Group label
                    ctx.fillStyle = 'black';
                    ctx.textAlign = 'center';
                    ctx.font = 'bold 14px Arial';
                    ctx.fillText(`Group ${group.id}`, (minX + maxX) / 2, minY - 20);
                }
            });
        }
        
        // Generate a color for a group
        function getGroupColor(groupId, alpha = 1) {
            const colors = [
                `rgba(255, 99, 132, ${alpha})`,   // Red
                `rgba(54, 162, 235, ${alpha})`,   // Blue
                `rgba(255, 206, 86, ${alpha})`,   // Yellow
                `rgba(75, 192, 192, ${alpha})`,   // Teal
                `rgba(153, 102, 255, ${alpha})`,  // Purple
                `rgba(255, 159, 64, ${alpha})`,   // Orange
                `rgba(199, 199, 199, ${alpha})`,  // Gray
                `rgba(83, 102, 255, ${alpha})`,   // Indigo
                `rgba(255, 99, 255, ${alpha})`,   // Pink
            ];
            
            return colors[groupId % colors.length];
        }

        // Generate random graph
        function generateRandomGraph(nodeCount = 10, edgeDensity = 0.3) {
            const graph = new Graph();
            
            // Add nodes
            for (let i = 0; i < nodeCount; i++) {
                graph.addNode(i, `Node ${i}`);
            }
            
            // Add edges (ensuring it's a DAG for Sugiyama to work properly)
            for (let i = 0; i < nodeCount; i++) {
                for (let j = i + 1; j < nodeCount; j++) {
                    if (Math.random() < edgeDensity) {
                        graph.addEdge(i, j);
                    }
                }
            }
            
            return graph;
        }
        
        // Update the groups UI
        function updateGroupsUI(graph) {
            const container = document.getElementById('groupsList');
            container.innerHTML = '';
            
            if (graph.groups.length === 0) {
                container.innerHTML = '<p>No groups defined yet. Click "Add Group" to create a group.</p>';
                return;
            }
            
            graph.groups.forEach(group => {
                const groupDiv = document.createElement('div');
                groupDiv.className = 'group-item';
                groupDiv.style.borderLeft = `5px solid ${getGroupColor(group.id)}`;
                
                const nodes = group.nodes
                    .map(id => graph.getNode(id))
                    .filter(node => node !== undefined)
                    .map(node => node.label)
                    .join(', ');
                
                groupDiv.innerHTML = `
                    <strong>Group ${group.id}</strong>: 
                    ${nodes}
                `;
                
                container.appendChild(groupDiv);
            });
        }
        
        // Create a new group dialog
        function showAddGroupDialog(graph) {
            // Create checkboxes for each node
            let checkboxesHtml = '';
            graph.nodes.forEach(node => {
                if (node.group === null) { // Only show ungrouped nodes
                    checkboxesHtml += `
                        <div>
                            <input type="checkbox" id="node-${node.id}" value="${node.id}">
                            <label for="node-${node.id}">${node.label}</label>
                        </div>
                    `;
                }
            });
            
            // Create dialog
            const dialog = document.createElement('div');
            dialog.style.position = 'fixed';
            dialog.style.left = '50%';
            dialog.style.top = '50%';
            dialog.style.transform = 'translate(-50%, -50%)';
            dialog.style.backgroundColor = 'white';
            dialog.style.padding = '20px';
            dialog.style.border = '1px solid #ccc';
            dialog.style.boxShadow = '0 4px 8px rgba(0,0,0,0.2)';
            dialog.style.zIndex = '1000';
            dialog.style.maxHeight = '80vh';
            dialog.style.overflow = 'auto';
            dialog.style.width = '400px';
            
            dialog.innerHTML = `
                <h3>Create New Group</h3>
                <p>Select nodes to include in the group:</p>
                <div style="margin-bottom: 15px;">
                    ${checkboxesHtml || '<p>No nodes available for grouping</p>'}
                </div>
                <div style="text-align: right;">
                    <button id="cancelAddGroup">Cancel</button>
                    <button id="confirmAddGroup">Create Group</button>
                </div>
            `;
            
            document.body.appendChild(dialog);
            
            // Add backdrop
            const backdrop = document.createElement('div');
            backdrop.style.position = 'fixed';
            backdrop.style.left = '0';
            backdrop.style.top = '0';
            backdrop.style.width = '100%';
            backdrop.style.height = '100%';
            backdrop.style.backgroundColor = 'rgba(0,0,0,0.5)';
            backdrop.style.zIndex = '999';
            document.body.appendChild(backdrop);
            
            // Add event listeners
            document.getElementById('cancelAddGroup').addEventListener('click', () => {
                document.body.removeChild(dialog);
                document.body.removeChild(backdrop);
            });
            
            document.getElementById('confirmAddGroup').addEventListener('click', () => {
                const selectedNodes = [];
                graph.nodes.forEach(node => {
                    const checkbox = document.getElementById(`node-${node.id}`);
                    if (checkbox && checkbox.checked) {
                        selectedNodes.push(node.id);
                    }
                });
                
                if (selectedNodes.length > 0) {
                    const groupId = graph.groups.length;
                    graph.createGroup(groupId, selectedNodes);
                    updateGroupsUI(graph);
                    
                    // Update layout and render
                    const layout = new SugiyamaLayout(graph);
                    graph = layout.run();
                    renderGraph(graph, document.getElementById('graphCanvas'));
                }
                
                document.body.removeChild(dialog);
                document.body.removeChild(backdrop);
            });
        }

        // Main code
        document.addEventListener('DOMContentLoaded', () => {
            const canvas = document.getElementById('graphCanvas');
            let graph = null;
            
            document.getElementById('generateGraph').addEventListener('click', () => {
                graph = generateRandomGraph();
                
                // Assign initial random positions
                graph.nodes.forEach(node => {
                    node.x = Math.random() * 700 + 50;
                    node.y = Math.random() * 500 + 50;
                });
                
                renderGraph(graph, canvas);
                updateGroupsUI(graph);
            });
            
            document.getElementById('addGroup').addEventListener('click', () => {
                if (!graph) {
                    alert('Please generate a graph first!');
                    return;
                }
                
                showAddGroupDialog(graph);
            });
            
            document.getElementById('layoutGraph').addEventListener('click', () => {
                if (!graph) {
                    alert('Please generate a graph first!');
                    return;
                }
                
                const layout = new SugiyamaLayout(graph);
                graph = layout.run();
                renderGraph(graph, canvas);
            });
        });
    </script>
</body>
</html>
```
노드 그룹화 기능을 성공적으로 구현했습니다. 이제 그래프의 노드들을 그룹화하고, 그룹화된 노드들이 같은 가로 범위 내에 배치되도록 제약 조건을 적용했습니다.

주요 변경 사항:

1. **노드 그룹 관리**
   - Graph 클래스에 그룹 관리 기능 추가 (createGroup, getNodesInGroup, getGroup)
   - 노드 객체에 그룹 속성 추가

2. **레이아웃 알고리즘 개선**
   - 같은 그룹의 노드들이 같은 계층 내에서 인접하도록 보장하는 기능 (ensureGroupsAdjacentInLayers)
   - 그룹의 가로 범위를 계산하는 함수 (calculateGroupBoundaries)
   - 그룹화된 노드들이 같은 가로 범위 내에 위치하도록 조정하는 함수 (adjustNodePositionsForGroups)

3. **UI 요소 추가**
   - "Add Group" 버튼을 통한 노드 그룹화 기능
   - 그룹 목록 표시 영역
   - 그룹 생성 모달 다이얼로그

4. **시각화 개선**
   - 그룹별 배경 영역 표시
   - 그룹별 색상 구분
   - 그룹 레이블 표시

사용 방법:
1. "Generate Random Graph" 버튼으로 랜덤 그래프 생성
2. "Add Group" 버튼을 클릭하여 그룹화할 노드 선택
3. "Apply Layout" 버튼으로 수기아마 알고리즘을 적용하면 그룹화된 노드들이 같은 가로 범위 내에 배치됨



# 각 그룹의 위치는 겹칠 수 없어


그룹들이 겹치지 않도록 코드를 수정하겠습니다.

```html:graph_layout.html
// ... existing code ...
            // Step 3: Assign x, y coordinates
            assignCoordinates() {
                const nodeWidth = 80;
                const nodeHeight = 40;
                const layerHeight = 120;
                
                // First assign standard coordinates
                for (let i = 0; i < this.layers.length; i++) {
                    const layer = this.layers[i];
                    const layerWidth = layer.length * nodeWidth;
                    const startX = (800 - layerWidth) / 2;
                    
                    layer.forEach((node, index) => {
                        node.x = startX + index * nodeWidth + nodeWidth / 2;
                        node.y = i * layerHeight + 60;
                    });
                }
                
                // Calculate group boundaries
                this.calculateGroupBoundaries(nodeWidth);
                
                // Adjust group positions to avoid overlaps
                this.adjustGroupsToAvoidOverlaps(nodeWidth);
                
                // Adjust node positions to satisfy group constraints
                this.adjustNodePositionsForGroups(nodeWidth);
                
                return this;
            }
            
            // Calculate min and max X values for each group across all layers
            calculateGroupBoundaries(nodeWidth) {
                // Reset group boundaries
                this.graph.groups.forEach(group => {
                    group.minX = Infinity;
                    group.maxX = -Infinity;
                });
                
                // Calculate initial group boundaries
                this.graph.nodes.forEach(node => {
                    if (node.group !== null) {
                        const group = this.graph.getGroup(node.group);
                        if (group) {
                            group.minX = Math.min(group.minX, node.x - nodeWidth/2);
                            group.maxX = Math.max(group.maxX, node.x + nodeWidth/2);
                        }
                    }
                });
            }
            
            // New function: Adjust group positions to avoid overlaps
            adjustGroupsToAvoidOverlaps(nodeWidth) {
                if (this.graph.groups.length <= 1) return;
                
                // Sort groups by minX for easier overlap detection
                const sortedGroups = [...this.graph.groups].sort((a, b) => a.minX - b.minX);
                
                // Check and fix overlaps
                for (let i = 1; i < sortedGroups.length; i++) {
                    const currentGroup = sortedGroups[i];
                    const prevGroup = sortedGroups[i-1];
                    
                    // If current group overlaps with previous group
                    if (currentGroup.minX < prevGroup.maxX) {
                        // Calculate how much we need to shift this group
                        const requiredShift = prevGroup.maxX - currentGroup.minX + nodeWidth; // Add extra space
                        
                        // Shift current group and all groups to its right
                        for (let j = i; j < sortedGroups.length; j++) {
                            sortedGroups[j].minX += requiredShift;
                            sortedGroups[j].maxX += requiredShift;
                        }
                    }
                }
            }
            
            // Adjust node positions to keep grouped nodes in the same horizontal range
            adjustNodePositionsForGroups(nodeWidth) {
                // Create a copy of nodes to avoid modifying while iterating
                const nodesCopy = [...this.graph.nodes];
                
                // Adjust node positions based on their group's boundaries
                nodesCopy.forEach(node => {
                    if (node.group !== null) {
                        const group = this.graph.getGroup(node.group);
                        if (group) {
                            // Calculate group center
                            const groupCenter = (group.minX + group.maxX) / 2;
                            
                            // Find all nodes in same group and same layer
                            const nodesInSameLayerAndGroup = this.layers[node.layer].filter(
                                n => n.group === node.group
                            );
                            
                            if (nodesInSameLayerAndGroup.length > 0) {
                                // Calculate current center of these nodes
                                let currentCenter = 0;
                                nodesInSameLayerAndGroup.forEach(n => {
                                    currentCenter += n.x;
                                });
                                currentCenter /= nodesInSameLayerAndGroup.length;
                                
                                // Calculate adjustment needed
                                const adjustment = groupCenter - currentCenter;
                                
                                // Apply adjustment
                                nodesInSameLayerAndGroup.forEach(n => {
                                    n.x += adjustment;
                                });
                            }
                        }
                    }
                });
            }
// ... existing code ...
        // Render group backgrounds
        function renderGroups(graph, ctx) {
            graph.groups.forEach((group, index) => {
                // Find min/max x and y for the group
                let minX = Infinity, minY = Infinity;
                let maxX = -Infinity, maxY = -Infinity;
                
                graph.getNodesInGroup(group.id).forEach(node => {
                    minX = Math.min(minX, node.x - 30);
                    maxX = Math.max(maxX, node.x + 30);
                    minY = Math.min(minY, node.y - 30);
                    maxY = Math.max(maxY, node.y + 30);
                });
                
                if (minX < Infinity) {
                    // Draw group background
                    ctx.fillStyle = getGroupColor(group.id, 0.2);
                    ctx.strokeStyle = getGroupColor(group.id);
                    ctx.lineWidth = 2;
                    
                    const padding = 20; // Increased padding to make groups more distinct
                    ctx.beginPath();
                    ctx.roundRect(
                        minX - padding, 
                        minY - padding, 
                        maxX - minX + padding * 2, 
                        maxY - minY + padding * 2,
                        10
                    );
                    ctx.fill();
                    ctx.stroke();
                    
                    // Group label
                    ctx.fillStyle = 'black';
                    ctx.textAlign = 'center';
                    ctx.font = 'bold 14px Arial';
                    ctx.fillText(`Group ${group.id}`, (minX + maxX) / 2, minY - 20);
                    
                    // Update group boundaries for next layout
                    group.minX = minX - padding;
                    group.maxX = maxX + padding;
                }
            });
        }
// ... existing code ...
```
그룹들이 서로 겹치지 않도록 코드를 수정했습니다. 주요 변경 사항은 다음과 같습니다:

1. **그룹 간 겹침 방지 기능 추가**:
   - 새로운 `adjustGroupsToAvoidOverlaps` 메소드를 추가하여 그룹들이 서로 겹치지 않도록 조정
   - 그룹들을 minX 값으로 정렬한 후, 겹침이 발생하면 오른쪽에 있는 그룹들을 모두 오른쪽으로 이동

2. **노드 위치 조정 로직 개선**:
   - 그룹 중심을 기준으로 동일 계층에 있는 같은 그룹의 노드들 위치를 조정
   - 이전보다 더 정확하게 그룹 중심을 유지하면서 노드 위치 조정

3. **시각적 개선**:
   - 그룹 배경의 여백(padding)을 증가시켜 그룹 간 구분을 더 명확하게 함
   - 그룹 렌더링 후 그룹의 경계값(minX, maxX)을 업데이트하여 다음 레이아웃에서도 일관성 유지

이제 그래프의 각 그룹들은 서로 겹치지 않는 공간에 배치됩니다. 그룹이 많아질수록 전체 그래프의 가로 공간이 자동으로 확장되어, 모든 그룹이 시각적으로 구분됩니다.


# 선의 끝에 화살표를 추가해서 노드간의 디펜던시를 표시하면 좋겠어.


화살표는 이미 코드에 구현되어 있지만, 그 크기와 가시성을 개선하도록 수정하겠습니다.

```html:graph_layout.html
// ... existing code ...
        // Render the graph
        function renderGraph(graph, canvas) {
            const ctx = canvas.getContext('2d');
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Draw group backgrounds
            renderGroups(graph, ctx);
            
            // Draw edges
            graph.edges.forEach(edge => {
                const sourceNode = graph.getNode(edge.source);
                const targetNode = graph.getNode(edge.target);
                
                // Calculate arrow endpoint near the target node circle
                const nodeRadius = 20;
                const dx = targetNode.x - sourceNode.x;
                const dy = targetNode.y - sourceNode.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                const directionX = dx / distance;
                const directionY = dy / distance;
                
                // Set endpoint at the edge of the target node's circle
                const endX = targetNode.x - directionX * nodeRadius;
                const endY = targetNode.y - directionY * nodeRadius;
                
                // Draw the line
                ctx.beginPath();
                ctx.moveTo(sourceNode.x, sourceNode.y);
                ctx.lineTo(endX, endY);
                ctx.strokeStyle = '#444'; // Darker color for better visibility
                ctx.lineWidth = 2;      // Increased line width
                ctx.stroke();
                
                // Draw arrow
                const arrowSize = 14;  // Increased arrow size
                const angle = Math.atan2(dy, dx);
                
                ctx.beginPath();
                ctx.moveTo(endX, endY);
                ctx.lineTo(
                    endX - arrowSize * Math.cos(angle - Math.PI/6),
                    endY - arrowSize * Math.sin(angle - Math.PI/6)
                );
                ctx.lineTo(
                    endX - arrowSize * Math.cos(angle + Math.PI/6),
                    endY - arrowSize * Math.sin(angle + Math.PI/6)
                );
                ctx.closePath();
                ctx.fillStyle = '#444'; // Match line color
                ctx.fill();
            });
            
            // Draw nodes
            graph.nodes.forEach(node => {
                // Node color: different if grouped
                ctx.fillStyle = node.group !== null ? getGroupColor(node.group) : '#3498db';
                ctx.beginPath();
                ctx.arc(node.x, node.y, 20, 0, Math.PI * 2);
                ctx.fill();
                
                // Add node border for better contrast with arrows
                ctx.strokeStyle = '#fff';
                ctx.lineWidth = 2;
                ctx.stroke();
                
                ctx.fillStyle = 'white';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.font = '12px Arial';
                ctx.fillText(node.label, node.x, node.y);
            });
        }
// ... existing code ...
```
화살표가 더 명확하게 보이도록 개선했습니다. 주요 변경 사항은 다음과 같습니다:

1. **화살표 계산 방식 개선**:
   - 선이 노드 중심이 아닌 노드 경계에서 시작하고 끝나도록 계산 로직 추가
   - 노드 테두리까지의 정확한 위치를 계산하여 화살표가 노드 안으로 파고들지 않게 함

2. **가시성 향상**:
   - 화살표 크기 증가 (10 → 14)
   - 선과 화살표의 색상을 더 진한 색('#999' → '#444')으로 변경
   - 선의 두께 증가 (1 → 2)

3. **화살표 모양 개선**:
   - 더 깔끔하고 전형적인 화살표 모양으로 수정
   - closePath()를 사용하여 깔끔한 삼각형 모양의 화살표 생성

4. **노드 테두리 추가**:
   - 흰색 테두리를 추가하여 화살표와 노드 사이의 경계를 더 명확하게 함

이제 노드 간 의존성 관계가 훨씬 더 명확하게 표시됩니다. 화살표가 노드 경계에서 끝나도록 함으로써 시각적으로 더 정확하고 보기 좋은 그래프가 되었습니다.
