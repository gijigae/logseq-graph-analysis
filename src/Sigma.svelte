<script lang="ts">
  import {
    onMount,
    onDestroy,
    createEventDispatcher,
    afterUpdate,
  } from "svelte";
  import Sigma from "sigma";
  import FA2Layout from "graphology-layout-forceatlas2/worker";
  import forceAtlas2 from "graphology-layout-forceatlas2";
  import type Graph from "graphology";
  import type { SigmaNodeEventPayload } from "sigma/sigma";
  import type { Attributes } from "graphology-types";
  import type { EdgeDisplayData, NodeDisplayData } from "sigma/types";
  import { Mode, settings } from "./stores";
  import { dijkstra, edgePathFromNodePath } from "graphology-shortest-path";
  import { adamicAdar, ResultMap } from "./analysis";

  export let graph: Graph;

  let sigmaRef: HTMLElement;
  let sigma: Sigma | undefined;
  let fa2Layout: FA2Layout | undefined;

  const dispatch = createEventDispatcher();

  const red = "#f87171";
  const orange = "#fb923c";
  const grey = "#a3a3a3";

  onMount(async () => {
    const g = graph;
    sigma = new Sigma(g, sigmaRef, {
      allowInvalidContainer: true,
      nodeReducer,
      edgeReducer,
      defaultNodeColor: grey,
    });
    sigma.on("clickNode", handleNodeClick);
  });

  afterUpdate(() => {
    if (sigma) {
      sigma.refresh();
      if (fa2Layout) {
        fa2Layout.kill();
      }
      const sensibleSettings = forceAtlas2.inferSettings(graph);
      sensibleSettings.gravity = $settings.gravity;
      fa2Layout = new FA2Layout(graph, {
        settings: sensibleSettings,
      });
      fa2Layout.start();
    }
  });

  onDestroy(async () => {
    if (fa2Layout) {
      fa2Layout.kill();
      fa2Layout = undefined;
    }
    if (sigma) {
      sigma.kill();
      sigma = undefined;
    }
  });

  const handleNodeClick = async ({ node }: SigmaNodeEventPayload) => {
    dispatch("nodeclick", node);
  };

  const nodeReducer = (node: string, data: Attributes) => {
    const res: Partial<NodeDisplayData> = { ...data };
    if ($settings.size === "in") {
      res.size = Math.max(1, graph.neighbors(node).length / 2);
    } else if ($settings.size === "out") {
      res.size = graph.inDegree(node);
    }

    if ($settings.search && res.label && res.label.includes($settings.search)) {
      res.color = orange;
    }

    if ($settings.mode === Mode.ShortestPath) {
      const pathA = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathA
      );
      const pathB = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathB
      );

      if (
        shortestNodePath?.includes(node) ||
        node === pathA ||
        node === pathB
      ) {
        res.color = red;
        res.zIndex = 2;
        res.highlighted = true;
      } else {
        res.zIndex = -1;
      }
    }

    if ($settings.mode === Mode.AdamicAdar && adamicAdarResults) {
      const pathA = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathA
      );
      if (pathA && node === pathA) {
        res.size = 10;
        res.zIndex = 2;
        res.color = orange;
      } else if (adamicAdarResults[node]) {
        res.color = red;
        res.size = 5 * adamicAdarResults[node].measure;
        res.label = `${adamicAdarResults[node].measure} ${data.label}`;
      } else {
        res.zIndex = -1;
        res.size = 1;
      }
    }

    return res;
  };

  let shortestNodePath: Array<string> | undefined | null;
  let shortestEdgePath: Array<string> | undefined | null;
  let adamicAdarResults: ResultMap | undefined;
  $: {
    if (
      sigma &&
      $settings.mode === Mode.ShortestPath &&
      $settings.pathA &&
      $settings.pathB
    ) {
      const graph = sigma.getGraph();
      const pathA = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathA
      );
      const pathB = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathB
      );
      if (pathA && pathB) {
        shortestNodePath =
          dijkstra.bidirectional(graph, pathA, pathB) ||
          dijkstra.bidirectional(graph, pathB, pathA);
        if (shortestNodePath) {
          shortestEdgePath = edgePathFromNodePath(graph, shortestNodePath);
        } else {
          shortestNodePath = undefined;
          shortestEdgePath = undefined;
        }
      } else {
        shortestNodePath = undefined;
        shortestEdgePath = undefined;
      }
    } else {
      shortestNodePath = undefined;
      shortestEdgePath = undefined;
    }

    if (sigma && $settings.mode === Mode.AdamicAdar && $settings.pathA) {
      const pathA = graph.findNode(
        (_, attrs) => attrs.label === $settings.pathA
      );
      if (pathA) {
        adamicAdarResults = adamicAdar(sigma.getGraph(), pathA);
        console.log(adamicAdarResults);
      } else {
        adamicAdarResults = undefined;
      }
    } else {
      adamicAdarResults = undefined;
    }
  }

  const edgeReducer = (edge: string, data: Attributes) => {
    const res: Partial<EdgeDisplayData> = { ...data };
    if (
      sigma &&
      $settings.mode === Mode.ShortestPath &&
      shortestEdgePath?.includes(edge)
    ) {
      res.color = red;
      res.size = 5;
    }
    return res;
  };
</script>

<div class="sigma" bind:this={sigmaRef} />

<style>
  .sigma {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
    overflow: hidden;
  }
</style>
