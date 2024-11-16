<script>
  import { onMount } from "svelte";
  import * as joint from "@joint/core";
  import Konva from "konva";
  import { writable } from "svelte/store";

  let stage;
  let layer;
  let graph;
  let paper;
  let container;

  const sourceElement = writable(null);

  onMount(() => {
    graph = new joint.dia.Graph();

    paper = new joint.dia.Paper({
      el: container,
      model: graph,
      width: 800,
      height: 600,
      gridSize: 10,
      drawGrid: true,
      background: {
        color: "#f8f9fa",
      },
      interactive: {
        linkMove: true,
        removeLinks: false,
      },
    });

    const Amp = joint.shapes.standard.Rectangle.define("audio.Amp", {
      attrs: {
        body: {
          fill: "#3498db",
          stroke: "#2980b9",
          strokeWidth: 2,
          rx: 5,
          ry: 5,
        },
        label: {
          text: "Amp",
          fill: "white",
          fontSize: 14,
        },
      },
    });

    const Mic = joint.shapes.standard.Circle.define("audio.Mic", {
      attrs: {
        body: {
          fill: "#e74c3c",
          stroke: "#c0392b",
          strokeWidth: 2,
        },
        label: {
          text: "Mic",
          fill: "white",
          fontSize: 14,
        },
      },
    });

    const amp = new Amp({
      position: { x: 100, y: 100 },
      size: { width: 80, height: 80 },
    });

    const mic = new Mic({
      position: { x: 300, y: 100 },
      size: { width: 60, height: 60 },
    });

    graph.addCells([amp, mic]);

    // Enable component dragging
    paper.on("element:pointerdown", function (elementView) {
      const element = elementView.model;

      paper.on("mousemove", function (evt) {
        const coords = paper.clientToLocalPoint(evt.clientX, evt.clientY);
        element.position(
          coords.x - element.get("size").width / 2,
          coords.y - element.get("size").height / 2
        );
      });

      paper.on("mouseup", function () {
        paper.off("mousemove");
        paper.off("mouseup");
      });
    });

    let currentSource = null;
    sourceElement.subscribe((value) => {
      currentSource = value;
    });

    // Modified linking logic with toggle functionality
    paper.on("element:pointerclick", function (elementView) {
      const selectedElement = elementView.model;

      if (!currentSource) {
        // Start linking mode
        sourceElement.set(selectedElement);
        selectedElement.attr("body/stroke", "#27ae60");
      } else if (currentSource.id === selectedElement.id) {
        // Cancel linking mode if clicking the same element
        selectedElement.attr(
          "body/stroke",
          selectedElement.original_stroke || selectedElement.attr("body/stroke")
        );
        sourceElement.set(null);
      } else {
        // Create link if clicking a different element
        const link = new joint.shapes.standard.Link({
          source: { id: currentSource.id },
          target: { id: selectedElement.id },
          attrs: {
            line: {
              stroke: "#34495e",
              strokeWidth: 2,
              targetMarker: {
                type: "path",
                stroke: "#34495e",
                fill: "#34495e",
                d: "M 10 -5 0 0 10 5 Z",
              },
            },
          },
        });

        graph.addCell(link);

        // Reset source element highlight and clear selection
        currentSource.attr(
          "body/stroke",
          currentSource.original_stroke || currentSource.attr("body/stroke")
        );
        sourceElement.set(null);
      }
    });

    // Store original stroke colors when elements are added
    graph.on("add", function (cell) {
      if (cell.isElement()) {
        cell.original_stroke = cell.attr("body/stroke");
      }
    });

    // Clear source selection on background click
    paper.on("blank:pointerclick", function () {
      if (currentSource) {
        currentSource.attr(
          "body/stroke",
          currentSource.original_stroke || currentSource.attr("body/stroke")
        );
        sourceElement.set(null);
      }
    });

    // Link removal
    paper.on("link:pointerclick", function (linkView, evt) {
      const tools = new joint.dia.ToolsView({
        tools: [
          new joint.linkTools.Remove({
            distance: "50%",
            action: function () {
              linkView.model.remove();
            },
          }),
        ],
      });

      linkView.addTools(tools);

      linkView.once("link:mouseleave", function () {
        linkView.removeTools();
      });
    });

    // Link highlighting
    paper.on("link:mouseenter", function (linkView) {
      linkView.model.attr({
        line: {
          strokeWidth: 4,
          stroke: "#FF4136",
        },
      });
    });

    paper.on("link:mouseleave", function (linkView) {
      linkView.model.attr({
        line: {
          strokeWidth: 2,
          stroke: "#34495e",
        },
      });
    });
  });
</script>

<div
  bind:this={container}
  style="width: 800px; height: 600px; border: 1px solid #ccc;"
></div>

<style>
  :global(.joint-element) {
    cursor: move;
  }

  :global(.joint-link) {
    cursor: pointer;
  }

  :global(.joint-link:hover) {
    cursor: pointer;
  }

  :global(.tool-remove) {
    cursor: pointer;
  }
</style>
