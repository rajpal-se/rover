<template>
  <fieldset>
    <legend>Graph</legend>
    <div id="cytoscape-container" ref="cy"></div>
  </fieldset>
</template>
>

<script>
import cytoscape from "cytoscape";
import klay from "cytoscape-klay";
import config from "./config";
import svg from "cytoscape-svg";
import { saveAs } from "file-saver";
import nodeHtmlLabel from "cytoscape-node-html-label";
import axios from "axios";

cytoscape.use(klay);

// Only load only once
if (typeof cytoscape("core", "svg") !== "function") {
  cytoscape.use(svg);
}

// Only load nodeHtmlLabel once
if (typeof cytoscape("core", "nodeHtmlLabel") !== "function") {
  cytoscape.use(nodeHtmlLabel);
}

export default {
  name: "Graph2",

  data: function () {
    return {
      graphData: {},
      isCtrlKeyActive: false,
    };
  },
  methods: {
    updateGraphData(data) {
      this.graphData = data;
      console.log("Data updated");
      setTimeout(() => {
        if (this.isCtrlKeyActive) {
          this.enableSelectionMode();
        } else {
          this.disableSelectionMode();
        }
      }, 0);
    },

    enableSelectionMode() {
      const { cy } = this;
      console.log("Enable");
      cy.autoungrabify(false);
      cy.autounselectify(false);
      cy.elements().unpanify();
      cy.elements().grabify();
    },

    disableSelectionMode() {
      const { cy } = this;
      console.log("Disable");
      cy.autoungrabify(true);
      cy.autounselectify(true);
      cy.elements().panify();
      cy.elements().ungrabify();
    },

    ctrlKeyHandler(e) {
      if (e.key === "Control") {
        if (e.type === "keyup") {
          this.isCtrlKeyActive = false;
        } else if (e.type === "keydown") {
          this.isCtrlKeyActive = true;
        }
      }
    },

    resetZoom() {
      const { cy } = this;
      cy.zoom(1);
      cy.fit();
      cy.layout(config.layout).run();
    },

    saveGraph: function () {
      const { cy } = this;
      const svgContent = cy.svg({ scale: 0.1, full: true });
      const blob = new Blob([svgContent], {
        type: "image/svg+xml;charset=utf-8",
      });
      saveAs(blob, "rover.svg");
    },
  },
  watch: {
    graphData: function (data) {
      const { cy } = this;

      // Remove Elements (node and edges)
      cy.elements().remove();

      // Add nodes
      cy.add(data.nodes);

      // Add edges
      const nodesNames = data.nodes.map((x) => x.data.id);
      data.edges.forEach((n) => {
        if (n.data.id.includes("-variable") || n.data.id.includes("-output")) {
          return;
        }

        // Browse node ancestors if target is not found in nodes
        // e.g: resource.test.attribute will not be in nodes, as attributes are not exported
        // this ensures that the dependency will be made on resource.test instead
        let name = n.data.target;
        while (!nodesNames.includes(name)) {
          name = name.split(".");
          if (name.length < 2) {
            console.warn("edge target", n.data.target, "not found in nodes");
            return;
          }
          name.pop();
          name = name.join(".");
        }
        n.data.target = name;

        // Add edge to the final graph
        cy.add(n);
      });

      // Rerender the layout
      cy.fit();
      cy.layout(config.layout).run();
    },

    isCtrlKeyActive: function (isActive) {
      console.log("IS ctrl ket active", isActive);
      if (isActive) {
        this.enableSelectionMode();
      } else {
        this.disableSelectionMode();
      }
    },
  },
  mounted: function () {
    const thisRef = this;
    axios.get(`/api/graph`).then((response) => {
      thisRef.updateGraphData(response.data);
      console.log(thisRef.isCtrlKeyActive);
    });

    /**
     * Setup Cytoscape
     */
    const cy = cytoscape({
      container: this.$refs.cy,
      style: config.style,
      wheelSensitivity: config.wheelSensitivity,
    });
    this.cy = cy;
    cy.boxSelectionEnabled(false);
    cy.panningEnabled(true);

    // Initially disable selection mode
    thisRef.disableSelectionMode();

    // Bind events
    document.addEventListener("keydown", thisRef.ctrlKeyHandler);
    document.addEventListener("keyup", thisRef.ctrlKeyHandler);

    console.log(thisRef, cy);
  },
};
</script>

<style>
#cytoscape-container {
  min-height: 1000px;
  background-color: #f8f8f8 !important;
}

.node {
  width: 14em;
  font-size: 2em;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  text-align: center;
  padding: 0.5em 0.5em;
  border-radius: 0.25em;
  background-color: white;
  color: black;
  font-weight: bold;
  cursor: pointer;
  border: 5px solid lightgray;
}

.node:hover {
  transform: scale(1.02);
}

.resource-type {
  width: 20em;
  font-size: 2em;
  height: 100%;
}

.create {
  background-color: #28a745;
  color: white;
  font-weight: bold;
  border: 0;
}

.delete {
  /* background-color: #ffe9e9;
  border: 5px solid #e40707; */
  background-color: #e40707;
  color: white;
  font-weight: bold;
  border: 0;
}

.update {
  /* background-color: #e1f0ff;
  border: 5px solid #1d7ada; */
  background-color: #1d7ada;
  color: white;
  font-weight: bold;
  border: 0;
}

.replace {
  /* background-color: #fff7e0;
  border: 5px solid #ffc107; */
  background-color: #ffc107;
  color: black;
  font-weight: bold;
  border: 0;
}

.output {
  background-color: #fff7e0;
  border: 5px solid #ffc107;
  color: black;
  font-weight: bold;
}

.variable {
  background-color: #e1f0ff;
  border: 5px solid #1d7ada;
  color: black;
  font-weight: bold;
}

.data {
  background-color: #ffecec;
  border: 5px solid #dc477d;
  color: black;
  font-weight: bold;
}

.locals {
  background-color: black;
  color: white;
  font-weight: bold;
  border: 0;
}
</style>

<style scoped>
fieldset {
  margin-bottom: 2em;
}

.graph-enter-active,
.graph-leave-active,
.graph-enter-active legend,
.graph-leave-active legend {
  transition: all 0.2s ease;
  overflow: hidden;
}

.graph-enter,
.graph-leave-to,
.graph-enter legend,
.graph-leave-to legend {
  height: 0;
  padding: 0;
  margin: 0;
  opacity: 0;
}
</style>
