<style scoped>
table {
    width: 100%;
    border-spacing: 0;
}

table td,
table th {
    text-align: center;
}

table.tools th,
table.tools td {
    width: 20%;
}


</style>

<template>
    <v-card>

        <v-card-title class="py-2">
            <v-icon small>mdi-wrench</v-icon> Workspaces
            <v-spacer></v-spacer>
        </v-card-title>

        <v-card-text class="pa-0">
            <template>
                
                <table class="tools">
                    <thead>
                        <th class="pl-1">Workspace</th>
                        <th class="pl-1">X</th>
                        <th class="px-1">Y</th>
                        <th class="px-1">Z</th>
                    </thead>
                    <tbody>
                        <!-- Tools -->
                        <template v-for="(work, workIndex) in workspaces">
                            <tr :key="`div-tool-${workIndex}`" :class="{ [selectedToolClass] : (workIndex === move.workspaceNumber-1) }">
                                <a href="javascript:void(0)" @click="workspaceClick(workIndex)">
                                    <td class="pl-1">{{ workspaceNumber(workIndex) }}</td>
                                </a>
                                <td class="pl-1">{{ $display(work[0], 3) }}</td>
                                <td class="px-1">{{ $display(work[1], 3) }}</td>
                                <td class="px-1">{{ $display(work[2], 3) }}</td>
                            </tr>
                        </template>
                    </tbody>
                </table>

            </template>
        </v-card-text>

    </v-card>
</template>

<script>
'use strict'

import { mapState, mapGetters, mapActions } from 'vuex'

import { DisconnectedError } from '../../utils/errors.js'
import { isPrinting } from '../../store/machine/modelEnums.js'

export default {
    computed: {
        ...mapGetters(['isConnected', 'uiFrozen']),
        ...mapState('machine/model', [ 'move']),
        ...mapState('settings', ['darkTheme']),

        selectedToolClass() {
            return this.darkTheme ? 'grey darken-3' : 'blue lighten-5';
        },

        visibleAxes() {
            return this.move.axes.filter(axis => axis.visible);
        },
        workspaces(){

            if(this.move.axes.length !=3 ){
                return [];
            }

            var work = [];
            for (var i = 0; i < this.move.axes[0].workplaceOffsets.length; i++) {
                work.push( [
                    this.move.axes[0].workplaceOffsets[i],
                    this.move.axes[1].workplaceOffsets[i],
                    this.move.axes[2].workplaceOffsets[i]
                    ] );
            }
            
            return work;
        },

    },
    data() {
        return {
            waitingForCode: false,
        }
    },
    methods: {
        ...mapActions('machine', ['sendCode']),
        workspaceNumber(index){
            if( index < 6 ){
                return 'G'+(54+index);
            }else{
                return 'G59.'+(index%5);
            }
        },
        workspaceClick(index) {
            if(isPrinting(this.status) && !this.waitingForCode){
                return;
            }
            try {
                this.waitingForCode = true;
                this.sendCode(this.workspaceNumber(index));
            } catch (e) {
                if (!(e instanceof DisconnectedError)) {
                    this.$log('error', e.message);
                }
            }
            this.waitingForCode = false;            
        },
        workspaceEdit(work,axis){
            console.log(this,work,axis);
        }
    }
}
</script>
