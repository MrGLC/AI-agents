# Project 08: Multi-Page Dashboard with D3.js Visualizations

## Overview
Build a comprehensive multi-page ML analytics dashboard with advanced D3.js visualizations. This project demonstrates creating interactive, data-driven visualizations for ML metrics, model performance, feature importance, and prediction analytics using D3.js's powerful data binding and transformation capabilities.

## Learning Objectives
- Master D3.js for custom data visualizations
- Implement interactive charts with zoom, pan, and tooltips
- Create multi-page dashboard architecture
- Build real-time updating visualizations
- Handle large datasets efficiently with D3
- Implement brushing and linking between charts
- Create custom SVG-based ML visualizations
- Apply responsive design to D3 charts

## Difficulty Level
**Advanced** - Requires strong understanding of D3.js, data visualization principles, SVG, and JavaScript/TypeScript.

## Technical Stack
- **Framework**: React or Vue (for routing and state)
- **Visualization**: D3.js v7+
- **Routing**: React Router or Vue Router
- **Data Processing**: D3-array, D3-scale, D3-axis
- **State Management**: Redux or Pinia
- **API Client**: Axios
- **UI Framework**: Tailwind CSS or Material-UI
- **Date Handling**: D3-time
- **Testing**: Vitest, D3-testing

## Requirements

### Dashboard Pages
1. **Overview Dashboard**: KPIs, summary charts
2. **Model Performance**: Accuracy, loss, metrics over time
3. **Predictions Analytics**: Distribution, confidence analysis
4. **Feature Importance**: Bar charts, radar charts
5. **Data Quality**: Missing values, outliers, distributions
6. **Comparison View**: Multi-model comparison
7. **Real-time Monitor**: Live updating metrics
8. **Export/Reports**: Download visualizations

### Visualization Types
1. Line charts (time series)
2. Bar charts (categorical comparisons)
3. Scatter plots (feature relationships)
4. Heatmaps (confusion matrices, correlations)
5. Box plots (distribution analysis)
6. Radar charts (multi-metric comparison)
7. Sankey diagrams (data flow)
8. Tree maps (hierarchical data)
9. Force-directed graphs (model architecture)
10. Histograms (value distributions)

### Interactive Features
1. Zoom and pan capabilities
2. Brush selection for filtering
3. Tooltips with detailed information
4. Click interactions for drill-down
5. Linked charts (brushing updates others)
6. Animated transitions
7. Export to PNG/SVG
8. Responsive resizing

### Performance Requirements
1. Handle 10,000+ data points smoothly
2. Smooth 60 FPS animations
3. Responsive updates < 100ms
4. Efficient data processing
5. Canvas fallback for large datasets

## Step-by-Step Implementation

### Step 1: Project Setup

```bash
# Create React project
npx create-react-app ml-d3-dashboard --template typescript

cd ml-d3-dashboard

# Install dependencies
npm install d3 @types/d3
npm install react-router-dom
npm install axios
npm install @reduxjs/toolkit react-redux
npm install tailwindcss @tailwindcss/forms
npm install date-fns
npm install html2canvas # For exporting charts
```

### Step 2: D3 Hook Utilities

Create `src/hooks/useD3.ts`:

```typescript
import { useRef, useEffect } from 'react';
import * as d3 from 'd3';

export function useD3<T extends Element>(
  renderFn: (svg: d3.Selection<T, unknown, null, undefined>) => void,
  dependencies: any[]
) {
  const ref = useRef<T>(null);

  useEffect(() => {
    if (ref.current) {
      const svg = d3.select(ref.current);
      renderFn(svg);
    }

    return () => {
      if (ref.current) {
        d3.select(ref.current).selectAll('*').remove();
      }
    };
  }, dependencies);

  return ref;
}

export function useResizeObserver<T extends Element>(
  callback: (width: number, height: number) => void
) {
  const ref = useRef<T>(null);

  useEffect(() => {
    if (!ref.current) return;

    const resizeObserver = new ResizeObserver((entries) => {
      if (!entries || entries.length === 0) return;

      const { width, height } = entries[0].contentRect;
      callback(width, height);
    });

    resizeObserver.observe(ref.current);

    return () => {
      resizeObserver.disconnect();
    };
  }, [callback]);

  return ref;
}
```

### Step 3: Line Chart Component

Create `src/components/charts/LineChart.tsx`:

```typescript
import React, { useState } from 'react';
import * as d3 from 'd3';
import { useD3, useResizeObserver } from '../../hooks/useD3';

export interface DataPoint {
  timestamp: Date;
  value: number;
  label?: string;
}

interface LineChartProps {
  data: DataPoint[];
  title?: string;
  yAxisLabel?: string;
  color?: string;
  showGrid?: boolean;
  enableZoom?: boolean;
}

export function LineChart({
  data,
  title,
  yAxisLabel,
  color = '#4f46e5',
  showGrid = true,
  enableZoom = true,
}: LineChartProps) {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  const containerRef = useResizeObserver<HTMLDivElement>((width, height) => {
    setDimensions({ width, height });
  });

  const svgRef = useD3<SVGSVGElement>(
    (svg) => {
      if (dimensions.width === 0 || data.length === 0) return;

      const margin = { top: 20, right: 30, bottom: 50, left: 60 };
      const width = dimensions.width - margin.left - margin.right;
      const height = 400 - margin.top - margin.bottom;

      // Clear previous content
      svg.selectAll('*').remove();

      // Create scales
      const xScale = d3
        .scaleTime()
        .domain(d3.extent(data, (d) => d.timestamp) as [Date, Date])
        .range([0, width]);

      const yScale = d3
        .scaleLinear()
        .domain([0, d3.max(data, (d) => d.value) || 0])
        .nice()
        .range([height, 0]);

      // Create main group
      const g = svg
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

      // Add grid
      if (showGrid) {
        g.append('g')
          .attr('class', 'grid')
          .attr('opacity', 0.1)
          .call(
            d3
              .axisLeft(yScale)
              .tickSize(-width)
              .tickFormat(() => '')
          );
      }

      // Create line generator
      const line = d3
        .line<DataPoint>()
        .x((d) => xScale(d.timestamp))
        .y((d) => yScale(d.value))
        .curve(d3.curveMonotoneX);

      // Add line path
      const path = g
        .append('path')
        .datum(data)
        .attr('fill', 'none')
        .attr('stroke', color)
        .attr('stroke-width', 2)
        .attr('d', line);

      // Animate line drawing
      const pathLength = path.node()?.getTotalLength() || 0;
      path
        .attr('stroke-dasharray', `${pathLength} ${pathLength}`)
        .attr('stroke-dashoffset', pathLength)
        .transition()
        .duration(1000)
        .ease(d3.easeLinear)
        .attr('stroke-dashoffset', 0);

      // Add dots
      g.selectAll('.dot')
        .data(data)
        .join('circle')
        .attr('class', 'dot')
        .attr('cx', (d) => xScale(d.timestamp))
        .attr('cy', (d) => yScale(d.value))
        .attr('r', 0)
        .attr('fill', color)
        .transition()
        .delay((d, i) => i * 10)
        .duration(500)
        .attr('r', 4);

      // Add axes
      const xAxis = d3
        .axisBottom(xScale)
        .ticks(6)
        .tickFormat((d) => d3.timeFormat('%b %d')(d as Date));

      const yAxis = d3.axisLeft(yScale).ticks(6);

      g.append('g')
        .attr('transform', `translate(0,${height})`)
        .call(xAxis)
        .selectAll('text')
        .attr('transform', 'rotate(-45)')
        .style('text-anchor', 'end');

      g.append('g').call(yAxis);

      // Add Y-axis label
      if (yAxisLabel) {
        g.append('text')
          .attr('transform', 'rotate(-90)')
          .attr('y', -margin.left + 15)
          .attr('x', -height / 2)
          .attr('text-anchor', 'middle')
          .text(yAxisLabel);
      }

      // Tooltip
      const tooltip = d3
        .select('body')
        .append('div')
        .attr('class', 'tooltip')
        .style('position', 'absolute')
        .style('visibility', 'hidden')
        .style('background-color', 'rgba(0, 0, 0, 0.8)')
        .style('color', 'white')
        .style('padding', '8px')
        .style('border-radius', '4px')
        .style('font-size', '12px')
        .style('pointer-events', 'none');

      g.selectAll('.dot')
        .on('mouseover', function (event, d: any) {
          d3.select(this)
            .transition()
            .duration(200)
            .attr('r', 6)
            .attr('fill', d3.color(color)!.brighter(0.5).toString());

          tooltip
            .style('visibility', 'visible')
            .html(
              `
              <strong>${d.label || 'Value'}</strong><br/>
              ${d3.timeFormat('%Y-%m-%d %H:%M')(d.timestamp)}<br/>
              Value: ${d.value.toFixed(2)}
            `
            );
        })
        .on('mousemove', function (event) {
          tooltip
            .style('top', event.pageY - 10 + 'px')
            .style('left', event.pageX + 10 + 'px');
        })
        .on('mouseout', function () {
          d3.select(this).transition().duration(200).attr('r', 4).attr('fill', color);

          tooltip.style('visibility', 'hidden');
        });

      // Zoom behavior
      if (enableZoom) {
        const zoom = d3
          .zoom<SVGSVGElement, unknown>()
          .scaleExtent([0.5, 10])
          .extent([
            [0, 0],
            [width, height],
          ])
          .on('zoom', (event) => {
            const newXScale = event.transform.rescaleX(xScale);

            g.select<SVGPathElement>('path')
              .attr(
                'd',
                line.x((d) => newXScale(d.timestamp))
              );

            g.selectAll<SVGCircleElement, DataPoint>('.dot').attr('cx', (d) =>
              newXScale(d.timestamp)
            );

            g.select<SVGGElement>('.x-axis').call(xAxis.scale(newXScale) as any);
          });

        svg.call(zoom as any);
      }
    },
    [data, dimensions, color, showGrid, enableZoom, yAxisLabel]
  );

  return (
    <div ref={containerRef} className="w-full h-full">
      {title && <h3 className="text-lg font-semibold mb-2">{title}</h3>}
      <svg
        ref={svgRef}
        width={dimensions.width}
        height={400}
        style={{ display: 'block' }}
      />
    </div>
  );
}
```

### Step 4: Confusion Matrix Heatmap

Create `src/components/charts/ConfusionMatrix.tsx`:

```typescript
import React, { useState } from 'react';
import * as d3 from 'd3';
import { useD3, useResizeObserver } from '../../hooks/useD3';

interface ConfusionMatrixProps {
  data: number[][];
  labels: string[];
  title?: string;
}

export function ConfusionMatrix({ data, labels, title }: ConfusionMatrixProps) {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  const containerRef = useResizeObserver<HTMLDivElement>((width, height) => {
    setDimensions({ width, height });
  });

  const svgRef = useD3<SVGSVGElement>(
    (svg) => {
      if (dimensions.width === 0) return;

      const margin = { top: 80, right: 30, bottom: 60, left: 80 };
      const size = Math.min(dimensions.width, 500);
      const width = size - margin.left - margin.right;
      const height = size - margin.top - margin.bottom;

      svg.selectAll('*').remove();

      const g = svg
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

      const cellSize = width / labels.length;

      // Color scale
      const maxValue = d3.max(data.flat()) || 1;
      const colorScale = d3
        .scaleSequential(d3.interpolateBlues)
        .domain([0, maxValue]);

      // Create cells
      const cells = g
        .selectAll('.cell')
        .data(data.flatMap((row, i) => row.map((value, j) => ({ i, j, value }))))
        .join('rect')
        .attr('class', 'cell')
        .attr('x', (d) => d.j * cellSize)
        .attr('y', (d) => d.i * cellSize)
        .attr('width', cellSize)
        .attr('height', cellSize)
        .attr('fill', 'white')
        .attr('stroke', '#ccc')
        .attr('stroke-width', 1)
        .style('cursor', 'pointer')
        .on('mouseover', function (event, d) {
          d3.select(this).attr('stroke', '#000').attr('stroke-width', 2);

          tooltip
            .style('visibility', 'visible')
            .html(
              `
              <strong>Predicted: ${labels[d.j]}</strong><br/>
              <strong>Actual: ${labels[d.i]}</strong><br/>
              Count: ${d.value}
            `
            );
        })
        .on('mousemove', function (event) {
          tooltip
            .style('top', event.pageY - 10 + 'px')
            .style('left', event.pageX + 10 + 'px');
        })
        .on('mouseout', function () {
          d3.select(this).attr('stroke', '#ccc').attr('stroke-width', 1);
          tooltip.style('visibility', 'hidden');
        });

      // Animate cells
      cells
        .transition()
        .delay((d, i) => i * 20)
        .duration(500)
        .attr('fill', (d) => colorScale(d.value));

      // Add text values
      g.selectAll('.cell-text')
        .data(data.flatMap((row, i) => row.map((value, j) => ({ i, j, value }))))
        .join('text')
        .attr('class', 'cell-text')
        .attr('x', (d) => d.j * cellSize + cellSize / 2)
        .attr('y', (d) => d.i * cellSize + cellSize / 2)
        .attr('text-anchor', 'middle')
        .attr('dominant-baseline', 'middle')
        .attr('opacity', 0)
        .text((d) => d.value)
        .attr('fill', (d) => (d.value > maxValue / 2 ? 'white' : 'black'))
        .attr('font-size', `${cellSize / 4}px`)
        .attr('font-weight', 'bold')
        .transition()
        .delay((d, i) => i * 20 + 500)
        .duration(300)
        .attr('opacity', 1);

      // Add X-axis labels
      g.selectAll('.x-label')
        .data(labels)
        .join('text')
        .attr('class', 'x-label')
        .attr('x', (d, i) => i * cellSize + cellSize / 2)
        .attr('y', height + 20)
        .attr('text-anchor', 'middle')
        .text((d) => d)
        .attr('font-size', '12px');

      // Add Y-axis labels
      g.selectAll('.y-label')
        .data(labels)
        .join('text')
        .attr('class', 'y-label')
        .attr('x', -10)
        .attr('y', (d, i) => i * cellSize + cellSize / 2)
        .attr('text-anchor', 'end')
        .attr('dominant-baseline', 'middle')
        .text((d) => d)
        .attr('font-size', '12px');

      // Add axis titles
      g.append('text')
        .attr('x', width / 2)
        .attr('y', height + 45)
        .attr('text-anchor', 'middle')
        .text('Predicted')
        .attr('font-weight', 'bold');

      g.append('text')
        .attr('transform', 'rotate(-90)')
        .attr('x', -height / 2)
        .attr('y', -60)
        .attr('text-anchor', 'middle')
        .text('Actual')
        .attr('font-weight', 'bold');

      // Tooltip
      const tooltip = d3
        .select('body')
        .selectAll('.confusion-tooltip')
        .data([null])
        .join('div')
        .attr('class', 'confusion-tooltip')
        .style('position', 'absolute')
        .style('visibility', 'hidden')
        .style('background-color', 'rgba(0, 0, 0, 0.8)')
        .style('color', 'white')
        .style('padding', '8px')
        .style('border-radius', '4px')
        .style('font-size', '12px')
        .style('pointer-events', 'none')
        .style('z-index', '1000');
    },
    [data, labels, dimensions]
  );

  return (
    <div ref={containerRef} className="w-full">
      {title && <h3 className="text-lg font-semibold mb-4 text-center">{title}</h3>}
      <svg
        ref={svgRef}
        width={dimensions.width}
        height={Math.min(dimensions.width, 500)}
      />
    </div>
  );
}
```

### Step 5: Feature Importance Bar Chart

Create `src/components/charts/FeatureImportance.tsx`:

```typescript
import React, { useState } from 'react';
import * as d3 from 'd3';
import { useD3, useResizeObserver } from '../../hooks/useD3';

interface Feature {
  name: string;
  importance: number;
}

interface FeatureImportanceProps {
  features: Feature[];
  title?: string;
  topN?: number;
}

export function FeatureImportance({
  features,
  title,
  topN = 10,
}: FeatureImportanceProps) {
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  const containerRef = useResizeObserver<HTMLDivElement>((width, height) => {
    setDimensions({ width, height });
  });

  const svgRef = useD3<SVGSVGElement>(
    (svg) => {
      if (dimensions.width === 0) return;

      const margin = { top: 20, right: 30, bottom: 40, left: 150 };
      const width = dimensions.width - margin.left - margin.right;
      const height = 400 - margin.top - margin.bottom;

      svg.selectAll('*').remove();

      // Sort and get top N features
      const topFeatures = features
        .sort((a, b) => b.importance - a.importance)
        .slice(0, topN);

      // Create scales
      const xScale = d3
        .scaleLinear()
        .domain([0, d3.max(topFeatures, (d) => d.importance) || 1])
        .range([0, width]);

      const yScale = d3
        .scaleBand()
        .domain(topFeatures.map((d) => d.name))
        .range([0, height])
        .padding(0.2);

      // Color scale
      const colorScale = d3
        .scaleSequential(d3.interpolateViridis)
        .domain([0, topFeatures.length]);

      const g = svg
        .append('g')
        .attr('transform', `translate(${margin.left},${margin.top})`);

      // Add bars
      g.selectAll('.bar')
        .data(topFeatures)
        .join('rect')
        .attr('class', 'bar')
        .attr('x', 0)
        .attr('y', (d) => yScale(d.name)!)
        .attr('height', yScale.bandwidth())
        .attr('width', 0)
        .attr('fill', (d, i) => colorScale(i))
        .attr('rx', 4)
        .style('cursor', 'pointer')
        .on('mouseover', function (event, d) {
          d3.select(this).attr('opacity', 0.8);
        })
        .on('mouseout', function () {
          d3.select(this).attr('opacity', 1);
        })
        .transition()
        .duration(800)
        .delay((d, i) => i * 50)
        .attr('width', (d) => xScale(d.importance));

      // Add value labels
      g.selectAll('.label')
        .data(topFeatures)
        .join('text')
        .attr('class', 'label')
        .attr('x', (d) => xScale(d.importance) + 5)
        .attr('y', (d) => yScale(d.name)! + yScale.bandwidth() / 2)
        .attr('dominant-baseline', 'middle')
        .attr('opacity', 0)
        .text((d) => d.importance.toFixed(3))
        .attr('font-size', '11px')
        .attr('fill', '#333')
        .transition()
        .delay((d, i) => i * 50 + 800)
        .duration(300)
        .attr('opacity', 1);

      // Add axes
      const yAxis = d3.axisLeft(yScale);
      const xAxis = d3.axisBottom(xScale).ticks(5);

      g.append('g').call(yAxis).selectAll('text').attr('font-size', '11px');

      g.append('g')
        .attr('transform', `translate(0,${height})`)
        .call(xAxis);

      // Add X-axis label
      g.append('text')
        .attr('x', width / 2)
        .attr('y', height + 35)
        .attr('text-anchor', 'middle')
        .text('Importance Score')
        .attr('font-size', '12px')
        .attr('fill', '#666');
    },
    [features, dimensions, topN]
  );

  return (
    <div ref={containerRef} className="w-full">
      {title && <h3 className="text-lg font-semibold mb-2">{title}</h3>}
      <svg ref={svgRef} width={dimensions.width} height={400} />
    </div>
  );
}
```

### Step 6: Dashboard Layout

Create `src/pages/Dashboard.tsx`:

```typescript
import React, { useEffect, useState } from 'react';
import { LineChart } from '../components/charts/LineChart';
import { ConfusionMatrix } from '../components/charts/ConfusionMatrix';
import { FeatureImportance } from '../components/charts/FeatureImportance';
import axios from 'axios';

export function Dashboard() {
  const [accuracyData, setAccuracyData] = useState([]);
  const [confusionData, setConfusionData] = useState<number[][]>([]);
  const [featureData, setFeatureData] = useState([]);

  useEffect(() => {
    loadDashboardData();
  }, []);

  const loadDashboardData = async () => {
    // Mock data for demonstration
    const mockAccuracy = Array.from({ length: 30 }, (_, i) => ({
      timestamp: new Date(Date.now() - (29 - i) * 24 * 60 * 60 * 1000),
      value: 0.85 + Math.random() * 0.1,
      label: `Day ${i + 1}`,
    }));

    const mockConfusion = [
      [50, 2, 1],
      [3, 45, 2],
      [1, 1, 48],
    ];

    const mockFeatures = [
      { name: 'credit_score', importance: 0.45 },
      { name: 'income', importance: 0.28 },
      { name: 'age', importance: 0.15 },
      { name: 'employment_years', importance: 0.08 },
      { name: 'loan_amount', importance: 0.04 },
    ];

    setAccuracyData(mockAccuracy as any);
    setConfusionData(mockConfusion);
    setFeatureData(mockFeatures as any);
  };

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-7xl mx-auto">
        <h1 className="text-3xl font-bold mb-8">ML Model Dashboard</h1>

        <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
          {/* Accuracy Over Time */}
          <div className="bg-white p-6 rounded-lg shadow">
            <LineChart
              data={accuracyData}
              title="Model Accuracy Over Time"
              yAxisLabel="Accuracy"
              color="#10b981"
              enableZoom={true}
            />
          </div>

          {/* Feature Importance */}
          <div className="bg-white p-6 rounded-lg shadow">
            <FeatureImportance
              features={featureData}
              title="Feature Importance"
              topN={5}
            />
          </div>
        </div>

        {/* Confusion Matrix */}
        <div className="bg-white p-6 rounded-lg shadow">
          <ConfusionMatrix
            data={confusionData}
            labels={['Class A', 'Class B', 'Class C']}
            title="Confusion Matrix"
          />
        </div>
      </div>
    </div>
  );
}
```

## Expected Outputs

1. **Interactive Dashboard**:
   - Multiple visualization types
   - Smooth animations
   - Interactive tooltips
   - Zoom/pan capabilities

2. **Data Visualizations**:
   - Line charts for time series
   - Heatmaps for confusion matrices
   - Bar charts for feature importance
   - Custom D3 visualizations

3. **Performance**:
   - Smooth 60 FPS
   - Handle large datasets
   - Responsive design
   - Efficient updates

4. **User Experience**:
   - Intuitive navigation
   - Export functionality
   - Real-time updates
   - Mobile responsive

## Bonus Challenges

1. **Advanced Charts**: Sankey diagrams, force graphs
2. **Real-time Updates**: WebSocket data streaming
3. **Export**: PNG/SVG/PDF export
4. **Annotations**: Add custom annotations to charts
5. **Brushing**: Link multiple charts with brush selection
6. **3D Visualizations**: WebGL-based 3D charts
7. **Custom Legends**: Interactive legends with filtering
8. **Animation Controls**: Play/pause time-series animations

## Resources

- [D3.js Documentation](https://d3js.org/)
- [Observable](https://observablehq.com/@d3)
- [D3 Graph Gallery](https://d3-graph-gallery.com/)
- [D3 in Depth](https://www.d3indepth.com/)
- [React + D3](https://2019.wattenberger.com/blog/react-and-d3)

## Success Criteria

### Functionality (40%)
- [ ] Multiple chart types working
- [ ] Interactive features functional
- [ ] Data updates smoothly
- [ ] Export working
- [ ] Responsive design

### Code Quality (30%)
- [ ] Clean D3 code
- [ ] Reusable components
- [ ] TypeScript typing
- [ ] Efficient data processing
- [ ] Memory management

### Visualizations (20%)
- [ ] Clear, readable charts
- [ ] Appropriate chart selection
- [ ] Good color choices
- [ ] Accessible design
- [ ] Smooth animations

### Performance (10%)
- [ ] 60 FPS rendering
- [ ] Handle large datasets
- [ ] Fast updates
- [ ] Efficient DOM manipulation
- [ ] Optimized re-renders
