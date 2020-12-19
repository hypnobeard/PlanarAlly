<script lang="ts">
import { InvalidationMode, SyncMode } from "@/core/comm/types";
import { GlobalPoint, LocalPoint } from "@/game/geom";
import { layerManager } from "@/game/layers/manager";
import { Circle } from "@/game/shapes/variants/circle";
import { gameStore } from "@/game/store";
import Tool from "@/game/ui/tools/tool.vue";
import { l2g } from "@/game/units";
import Component from "vue-class-component";
import { ToolBasics } from "./ToolBasics";
import { ToolName, ToolPermission } from "./utils";
import { floorStore } from "../../layers/store";
import { sendShapePositionUpdate } from "../../api/emits/shape/core";
import { SelectFeatures } from "./select.vue";

@Component
export default class PingTool extends Tool implements ToolBasics {
    name = ToolName.Ping;
    active = false;
    startPoint: GlobalPoint | null = null;
    ping: Circle | null = null;
    border: Circle | null = null;

    get permittedTools(): ToolPermission[] {
        return [{ name: ToolName.Select, features: { enabled: [SelectFeatures.Context] } }];
    }

    cleanup(): void {
        if (!this.active || this.ping === null || this.border === null || this.startPoint === null) return;
        const layer = layerManager.getLayer(floorStore.currentFloor, "draw");
        if (layer === undefined) {
            console.log("No active layer!");
            return;
        }

        this.active = false;
        layer.removeShape(this.ping, SyncMode.TEMP_SYNC);
        layer.removeShape(this.border, SyncMode.TEMP_SYNC);
        this.ping = null;
        this.startPoint = null;
    }

    onDeselect(): void {
        this.cleanup();
    }

    onDown(lp: LocalPoint): void {
        this.cleanup();
        this.startPoint = l2g(lp);
        const layer = layerManager.getLayer(floorStore.currentFloor, "draw");

        if (layer === undefined) {
            console.log("No draw layer!");
            return;
        }

        this.active = true;
        this.ping = new Circle(this.startPoint, 20, { fillColour: gameStore.rulerColour });
        this.border = new Circle(this.startPoint, 40, { fillColour: "#0000", strokeColour: gameStore.rulerColour });
        this.ping.addOwner({ user: gameStore.username, access: { edit: true } }, false);
        this.border.addOwner({ user: gameStore.username, access: { edit: true } }, false);
        layer.addShape(this.ping, SyncMode.TEMP_SYNC, InvalidationMode.NORMAL);
        layer.addShape(this.border, SyncMode.TEMP_SYNC, InvalidationMode.NORMAL);
    }

    onUp(): void {
        this.cleanup();
    }

    onMove(lp: LocalPoint): void {
        if (!this.active || this.ping === null || this.border === null || this.startPoint === null) return;

        const gp = l2g(lp);

        const layer = layerManager.getLayer(floorStore.currentFloor, "draw");
        if (layer === undefined) {
            console.log("No draw layer!");
            return;
        }

        this.ping.center(gp);
        this.border.center(gp);

        sendShapePositionUpdate([this.ping, this.border], true);

        layer.invalidate(true);
    }
}
</script>