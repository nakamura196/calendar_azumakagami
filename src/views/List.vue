<template>
    <v-app>
        <v-content>
            <v-container class="my-2">
                <v-form ref="form" class="mb-2">
                    <v-layout row wrap>
                        <v-flex xs12 sm6>
                            <v-text-field v-model="q" label="キーワード / Keyword" name="q" class="mx-2"></v-text-field>
                        </v-flex>
                        <v-flex xs12 sm6>
                            <v-text-field v-model="date" name="date" label="年月 / Year and Month" class="mx-2"></v-text-field>
                        </v-flex>
    
                        <v-btn color="primary" @click="updatePath">
                            <i class="material-icons">search</i>
                        </v-btn>
                    </v-layout>
    
    
                </v-form>
    
                <FullCalendar class='mt-5 demo-app-calendar' ref="fullCalendar" defaultView="dayGridMonth" :header="{
                        left: 'prev,next',
                        center: 'title',
                        right: 'dayGridMonth,timeGridWeek,timeGridDay,listWeek'
                    }" :plugins="calendarPlugins" :weekends="calendarWeekends" :events="calendarEvents" @eventClick="handleDateClick" :defaultDate="defaultDate" />
            </v-container>
        </v-content>
    </v-app>
</template>

<script>
import FullCalendar from '@fullcalendar/vue'
import dayGridPlugin from '@fullcalendar/daygrid'
import timeGridPlugin from '@fullcalendar/timegrid'
import interactionPlugin from '@fullcalendar/interaction'
import axios from 'axios';

// str: 日付文字列（yyyy-MM-dd, yyyy/MM/dd）
// delim: 区切り文字（"-", "/"など）
function isDate(str, delim) {
    var arr = str.split(delim);
    if (arr.length !== 3) return false;
    const date = new Date(arr[0], arr[1] - 1, arr[2]);
    if (arr[0] !== String(date.getFullYear()) || arr[1] !== ('0' + (date.getMonth() + 1)).slice(-2) || arr[2] !== ('0' + date.getDate()).slice(-2)) {
        return false;
    } else {
        return true;
    }
};

function dateToYearAndMonth(date) {
    let year = date.getFullYear()
    let month = date.getMonth() + 1
    month = month < 10 ? "0" + month : month

    return year + "-" + month
}

export default {
    components: {
        FullCalendar // make the <FullCalendar> tag available
    },
    data: function() {
        return {
            calendarPlugins: [ // plugins must be defined in the JS
                dayGridPlugin,
                timeGridPlugin,
                interactionPlugin // needed for dateClick
            ],
            calendarWeekends: true,
            calendarEvents: [],
            defaultDate: new Date("1918-01"),
            q: "",
            type: "",
            collection: "",
            date: "1918-01",
            collection: ""
        }
    },
    watch: {
        '$route' (to, from) {
            this.search()
        }
    },
    created: function() {
        let param = Object.assign({}, this.$route.query)
        if (param.q) {
            this.q = param.q
        }
        if (param.type) {
            this.type = param.type
            //this.type_array = this.type.split(',');
        }
        if (param.collection) {
            this.collection = param.collection
            //this.type_array = this.type.split(',');
        }
        if (param.date) {
            this.defaultDate = new Date(param.date)
            this.date = dateToYearAndMonth(this.defaultDate)
        }
        this.search()
    },

    methods: {
        handleDateClick(arg) {
            window.open(arg.event.id, "_blank");
        },
        updatePath() {
            //this.type = this.type_array.join(',');
            this.defaultDate = new Date(this.date)
            this.$router.push({ path: "list", query: { q: this.q, type: this.type, date: this.date, collection: this.collection } })
        },
        search() {
            this.calendarEvents = []

            let query = " PREFIX dcterms: <http://purl.org/dc/terms/> \n";
            query += " PREFIX dcndl: <http://ndl.go.jp/dcndl/terms/> \n";
            query += " select distinct ?link ?date ?title \n";
            query += " WHERE { \n";
            if (this.type) {
                let type_array = this.type.split(",")
                for (let j = 0; j < type_array.length; j++) {
                    query += " { ?s dcndl:materialType <http://ndl.go.jp/ndltype/" + type_array[j] + "> . } \n";
                    if (j != type_array.length - 1) {
                        query += " UNION \n"
                    }
                }
            }
            if (this.collection) {
                let collection_array = this.collection.split(",")
                for (let j = 0; j < collection_array.length; j++) {
                    query += " { ?s dcterms:isPartOf '"+collection_array[j]+"'@ja . } \n";
                    if (j != collection_array.length - 1) {
                        query += " UNION \n"
                    }
                }
            }
            query += " ?s dcterms:date ?date . \n";
            query += " ?date bif:contains '\"" + this.date + "\"' . \n";
            query += " ?s dcterms:title ?title . \n";
            query += " ?s rdfs:seeAlso ?link . \n";
            if (this.q) {
                query += " ?s ?p ?o . \n";
                query += " ?o bif:contains '\"" + this.q + "\"' . \n";
            }

            query += " } order by desc(?date) \n";

            axios.get("https://nakamura196.github.io/jpsearch/data/azumakagami/data.json")
                .then(response => {

                    let result = response.data

                    let events = []

                    for (let i = 0; i < result.length; i++) {
                        let obj = result[i];

                        let date = obj["http://purl.org/dc/terms/date"][0]["@value"]

                        let title = obj["http://purl.org/dc/terms/title"][0]["@value"]

                        if (this.q) {
                            if(title.indexOf(this.q) == -1){
                                continue
                            }
                        }

                        if (isDate(date, '-')) {

                            let date2 = new Date(date)
                            let yearAndMonth = dateToYearAndMonth(date2)

                            let url = ""

                            let relations = obj["http://purl.org/dc/terms/relation"]

                            if(relations){
                                let tei = obj["http://purl.org/dc/terms/isPartOf"][0]["@id"]
                                url = "https://tei-eaj.github.io/aozora_tei/tools/visualization/facs/?url="+tei+"&textDirection=vertical&canvas="+relations[0]["@id"]
                            }

                            //if(yearAndMonth == this.date){
                                events.push({
                                    title: title,
                                    start: date,
                                    id: url
                                })
                            //}   
                        }
                    }

                    this.calendarEvents = events

                }).catch(error => { console.log(error); });
        },
        /*
        move(d) {
            let calendarApi = this.$refs.fullCalendar.getApi()
            if (d == -1) {
                calendarApi.prev()
            } else {
                calendarApi.next()
            }

            let currentDate = calendarApi.getDate()
            this.date = dateToYearAndMonth(currentDate)

            this.updatePath()

        }
        */
    }
}
</script>

<style lang='scss'>
// you must include each plugins' css
// paths prefixed with ~ signify node_modules
@import '~@fullcalendar/core/main.css';
@import '~@fullcalendar/daygrid/main.css';
@import '~@fullcalendar/timegrid/main.css';
</style>