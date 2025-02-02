<template>
    <div class="block">
        <!-- left and right panel buttons for dynamic panels -->
        <div class="flex">
            <button
                @click="() => changePanel('text')"
                class="border hover:bg-gray-100"
                :class="editingStatus === 'text' ? 'border-black' : 'border-gray-300'"
            >
                {{ $t('dynamic.textSection') }}
            </button>
            <button
                @click="() => changePanel('panels')"
                class="border hover:bg-gray-100"
                :class="editingStatus !== 'text' ? 'border-black' : 'border-gray-300'"
            >
                {{ $t('dynamic.panel.collection') }}
            </button>
        </div>
        <!-- Text Section -->
        <div v-if="editingStatus === 'text'">
            <component
                :is="'text-editor'"
                key="text"
                :panel="panel"
                :configFileStructure="configFileStructure"
                :lang="lang"
            ></component>
        </div>
        <div v-if="editingStatus === 'panels'">
            <table class="w-2/3 mt-5">
                <tr class="table-header">
                    <th>{{ $t('dynamic.panel.id') }}</th>
                    <th>{{ $t('dynamic.panel.type') }}</th>
                    <th>{{ $t('dynamic.panel.actions') }}</th>
                </tr>
                <tr class="table-contents" v-for="(item, idx) in panel.children" :key="idx">
                    <td>{{ item.id }}</td>
                    <td>{{ item.panel.type }}</td>
                    <td>
                        <span @click="() => switchSlide(idx)">{{ $t('editor.chart.label.edit') }}</span> |
                        <span @click="() => removeSlide(idx)">{{ $t('editor.remove') }}</span>
                    </td>
                </tr>
                <tr class="table-add-row">
                    <th class="flex flex-col items-center">
                        <input type="text" placeholder="Enter Panel ID" v-model="newSlideName" />
                        <p v-if="idUsed">{{ $t('dynamic.panel.idTaken') }}</p>
                    </th>
                    <th>
                        <select v-model="newSlideType">
                            <option
                                v-for="thing in Object.keys(editors).filter((editor) => editor !== 'image')"
                                :key="thing"
                            >
                                {{ thing }}
                            </option>
                        </select>
                    </th>
                    <th><button @click="createNewSlide" :disabled="idUsed">Add New</button></th>
                </tr>
            </table>

            <div v-if="editingSlide !== -1">
                <br />
                <hr />
                <br />
                <span class="font-bold text-xl">{{ $t('dynamic.panel.editor') }}</span
                ><br />
                <component
                    ref="slide"
                    :is="
                        editors[
                            panel.children[editingSlide].panel.type === 'image'
                                ? 'slideshow'
                                : panel.children[editingSlide].panel.type
                        ]
                    "
                    :key="editingSlide + panel.children[editingSlide].panel.type"
                    :panel="panel.children[editingSlide].panel"
                    :configFileStructure="configFileStructure"
                    :lang="lang"
                    :sourceCounts="sourceCounts"
                ></component>
            </div>
        </div>
    </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';
import {
    ChartConfig,
    ChartPanel,
    ConfigFileStructure,
    DefaultConfigs,
    DynamicChildItem,
    DynamicPanel,
    ImagePanel,
    MapPanel,
    PanelType,
    SlideshowPanel,
    SourceCounts
} from '@/definitions';

import ChartEditorV from './chart-editor.vue';
import ImageEditorV from './image-editor.vue';
import TextEditorV from './text-editor.vue';
import MapEditorV from './map-editor.vue';

@Component({
    components: {
        'chart-editor': ChartEditorV,
        'image-editor': ImageEditorV,
        'text-editor': TextEditorV,
        'dynamic-editor': DynamicEditorV,
        'map-editor': MapEditorV
    }
})
export default class DynamicEditorV extends Vue {
    @Prop() panel!: DynamicPanel;
    @Prop() configFileStructure!: ConfigFileStructure;
    @Prop() lang!: string;
    @Prop() sourceCounts!: SourceCounts;

    editors: Record<string, string> = {
        text: 'text-editor',
        image: 'image-editor',
        slideshow: 'image-editor',
        chart: 'chart-editor',
        map: 'map-editor'
    };

    startingConfig: DefaultConfigs = {
        text: {
            type: PanelType.Text,
            title: '',
            content: ''
        },
        dynamic: {
            type: PanelType.Dynamic,
            title: '',
            titleTag: '',
            content: '',
            children: []
        },
        slideshow: {
            type: PanelType.Slideshow,
            images: []
        },
        chart: {
            type: PanelType.Chart,
            charts: []
        },
        map: {
            type: PanelType.Map,
            config: '',
            title: '',
            scrollguard: false
        }
    };

    editingStatus = 'text';
    editingSlide = -1;

    newSlideName = '';
    newSlideType = 'text';

    get idUsed(): boolean {
        return this.panel.children.some((ch: DynamicChildItem) => ch.id === this.newSlideName);
    }

    changePanel(target: string): void {
        this.saveChanges();
        this.editingStatus = target;
    }

    switchSlide(idx: number): void {
        // Save slide changes if neccessary and switch to the newly selected slide.
        this.saveChanges();
        this.editingSlide = idx;

        // Image Panel to Slideshow Panel Conversion
        if (this.panel.children[this.editingSlide].panel.type === 'image') {
            (this.panel.children[this.editingSlide].panel as SlideshowPanel) = {
                type: PanelType.Slideshow,
                images: [this.panel.children[this.editingSlide].panel as ImagePanel]
            };
        }
    }

    removeSlide(item: number): void {
        const panel = this.panel.children.find((panel: DynamicChildItem, idx: number) => idx === item)?.panel;

        // Update source counts based on which panel is removed.
        switch (panel?.type) {
            case 'map': {
                const mapPanel = panel as MapPanel;
                this.sourceCounts[mapPanel.config] -= 1;
                if (this.sourceCounts[mapPanel.config] === 0) {
                    this.configFileStructure.zip.remove(
                        `${mapPanel.config.substring(mapPanel.config.indexOf('/') + 1)}`
                    );
                }
                break;
            }

            case 'chart': {
                const chartPanel = panel as ChartPanel;
                chartPanel.charts.forEach((chart: ChartConfig) => {
                    this.sourceCounts[chart.src] -= 1;
                    if (this.sourceCounts[chart.src] === 0) {
                        this.configFileStructure.zip.remove(`${chart.src.substring(chart.src.indexOf('/') + 1)}`);
                    }
                });
                break;
            }

            case 'slideshow': {
                const slideshowPanel = panel as SlideshowPanel;
                slideshowPanel.images.forEach((image: ImagePanel) => {
                    this.sourceCounts[image.src] -= 1;
                    if (this.sourceCounts[image.src] === 0) {
                        this.configFileStructure.zip.remove(`${image.src.substring(image.src.indexOf('/') + 1)}`);
                    }
                });
                break;
            }
        }

        // Remove the panel itself.
        this.panel.children = this.panel.children.filter((panel: DynamicChildItem, idx: number) => idx !== item);

        // If the slide being removed is the currently selected slide, unselect it.
        if (this.editingSlide === item) {
            this.editingSlide = -1;
        }
    }

    createNewSlide(): void {
        if (!this.newSlideName) return;

        const newConfig = {
            id: this.newSlideName,
            panel: JSON.parse(JSON.stringify(this.startingConfig[this.newSlideType as keyof DefaultConfigs]))
        };

        this.newSlideName = '';
        this.panel.children.push(newConfig);
    }

    saveChanges(): void {
        if (
            this.$refs.slide !== undefined &&
            typeof (this.$refs.slide as ImageEditorV | ChartEditorV).saveChanges === 'function'
        ) {
            (this.$refs.slide as ImageEditorV | ChartEditorV).saveChanges();
        }
    }
}
</script>

<style lang="scss">
.table-header th {
    text-align: center;
    background-color: #ddd;
    padding: 5px;
}
.table-contents td {
    text-align: center;
    padding: 5px;
}
.table-contents:hover {
    background-color: #eee;
    cursor: pointer;
}
.table-add-row th {
    vertical-align: top;
    text-align: center;
    border-top: 1px solid #ddd;
    padding: 5px;
}
.table-add-row input[type='text'],
.table-add-row select,
.table-add-row button {
    width: 150px !important;
    text-align: center;
    font-weight: normal;
    border: 1px solid black;
    padding: 2px !important;
    margin-top: 0 !important;
}
</style>
