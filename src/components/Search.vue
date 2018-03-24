<template>
<div class="container">
        <div class="row py-3">
            <div class="col-md-3">
                <div class="hpanel">
                    <div class="panel-body">
                        <div>
                            <h4>
                                Filters
                            </h4>
                        </div>

                        <div class="form-group">
                            <label class="control-label">Date:</label>
                            <div class="input-group date">
                                <date-picker name="filter-start-date" v-model="startDate" :config="config"></date-picker>
                                <span class="input-group-addon"><i class="glyphicon glyphicon-calendar"></i></span>
                            </div>
                            <div class="input-group">
                                ~
                            </div>
                            <div class="input-group date">
                                <date-picker name="filter-end-date" v-model="endDate" :config="config"></date-picker>
                                <span class="input-group-addon"><i class="glyphicon glyphicon-calendar"></i></span>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="control-label">Fields:</label>
                            <div class="input-group">
                                <div class="checkbox checkbox-primary" v-for="field in fields" >
                                    <input name="filter-fields" v-bind:value="field.value" type="checkbox" v-model="searchFields">
                                    <label for="checkbox1">
                                        {{ field.text }}
                                    </label>
                                </div>
                            </div>
                        </div>
                        <div class="form-group">
                            <label class="control-label">Sort:</label>
                            <div class="input-group">
                                <select class="form-control m-b" id="filter-sort" name="filter-sort" v-model="sort">
                                    <option v-for="option in options" v-bind:value="option.value"> {{ option.text }} </option>
                                </select>
                            </div>
                        </div>
                        
                        <div class="form-group" v-if="buckets.length > 0">
                            <label class="control-label">Category:</label>
                            <div class="input-group">
                              <div class="checkbox checkbox-primary" v-for="bucket in buckets">
                                  <input name="filter-categorys" v-bind:value="bucket.key" type="checkbox" v-model="category">
                                  <label for="checkbox1">
                                      {{ bucket.key }} ({{ bucket.doc_count }})
                                  </label>
                              </div>
                            </div>
                        </div>                        

                        <button class="btn btn-success btn-block" id="btn-apply" v-on:click="apply">Apply</button>
                    </div>
                </div>
            </div>
            <div class="col-md-9">
                <div class="row">
                    <div class="col-lg-12">
                        <div class="hpanel">
                            <div class="panel-body">
                                <form name="searchForm" id="searchForm" method="get" @submit.prevent="search">
                                    <div class="input-group">
                                        <input class="form-control" type="text" id="query" name="query" v-model="query" placeholder="Search.." >
                                        <div class="input-group-btn">
                                            <button type="button" class="btn btn-outline-primary" v-on:click="search">Search</button>
                                        </div>
                                    </div>
                                </form>
                                <div style="padding-top: 10px;">
                                    Search Results for <strong>{{ index }}</strong> ({{totalCount}} results)
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
                <div class="row">
                  <div class="col-lg-12">
                    <div class="hpanel filter-item" v-for="item in items">
                      <a href="#">
                        <div class="panel-body">
                          <div class="pull-right text-right">
                            <small class="stat-label">{{ item._source.date }}</small>
                          </div>
                          <h4 class="m-b-xs" v-if="typeof item.highlight !== 'undefined' && typeof item.highlight.title !== 'undefined'">
                            <span v-for="title in item.highlight.title">
                              <span v-if="typeof title !== 'undefined'" v-html="title"></span>
                              <span v-else v-html="item.highlight.title"></span>
                            </span>                            
                          </h4>
                          <h4 class="m-b-xs" v-else>{{ item._source.title }}</h4>
                          {{ item._source.author }} / {{ item._source.category }}
                          </p>
                          <p v-if="typeof item.highlight !== 'undefined' && typeof item.highlight.desc !== 'undefined'">
                            <span v-for="desc in item.highlight.desc">
                              <span v-if="typeof desc !== 'undefined'" v-html="desc"></span>
                              <span v-else v-html="item.highlight.desc"></span>
                            </span>                            
                          </p>
                          <p v-else>{{ item._source.desc }}</p>
                        </div>
                      </a>
                    </div>

                    <div class="text-center">
                      <div class="col-lg-12 py-3">
                        <ul class="pagination mx-auto justify-content-center">
                            <li class="page-item" v-bind:class="{ disabled: pageNum<=1 }">
                                <a class="page-link" href="javascript:void(0);" v-on:click="goPage(pageNum-1)" aria-label="Previous">
                                    <span aria-hidden="true">«</span>
                                    <span class="sr-only">Previous</span>
                                </a>
                            </li>
                            <li v-for="i in pageCount" class="page-item" v-bind:class="{ active: pageNum==i }">
                                <a class="page-link" href="javascript:void(0);" v-on:click="goPage(i)">{{ i }}</a>
                            </li>
                            <li class="page-item" v-bind:class="{ disabled: pageNum>=pageCount }">
                                <a class="page-link" href="javascript:void(0);" v-on:click="goPage(pageNum+1)" aria-label="Next">
                                    <span aria-hidden="true">»</span>
                                    <span class="sr-only">Next</span>
                                </a>
                            </li>
                        </ul>
                      </div>
                    </div>
                  </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import elasticsearch from 'elasticsearch'

const client = new elasticsearch.Client({
  host: [
    {
      host: '127.0.0.1',
      protocol: 'http',
      port: 9200
    }
  ]
})

export default {
  name: 'search',
  data () {
    return {
      query: '',
      index: 'blogs',
      type: 'blog',
      totalCount: 0,
      perPage: 10,
      pageNum: 1,
      pageCount: 1,
      startDate: '',
      endDate: '',
      config: {
        format: 'YYYY.MM.DD',
        useCurrent: false,
        showClear: true,
        showClose: true
      },
      sort: '_score',
      options: [
        { value: '_score', text: 'Score' },
        { value: 'date', text: 'Date' }
      ],
      fields: [
        { value: 'title', text: 'Title' },
        { value: 'desc', text: 'Desc' },
        { value: 'author', text: 'Author' }
      ],
      searchFields: ['title', 'desc', 'author'],
      buckets: [],
      items: [],
      category: []
    }
  },
  methods: {
    search: function () {
      let query = ''
      let date = ''
      let range = ''
      let must = ''
      let sorts = {}

      // Set sort query
      if (this.sort === 'date') {
        sorts = { 'date': 'desc' }
      } else {
        sorts = { '_score': 'desc' }
      }

      // Set date query
      if (this.startDate !== '' && this.endDate !== '') {
        date = { 'gte': this.startDate, 'lte': this.endDate }
      } else if (this.startDate !== '') {
        date = { 'gte': this.startDate }
      } else if (this.endDate !== '') {
        date = { 'lte': this.endDate }
      }

      // Set range query
      if (date !== '') {
        range = {'date': date}
      }

      // Set must query
      if (range !== '') {
        must = [{'range': range}, {'multi_match': {'query': this.query, 'fields': this.searchFields}}]
      } else {
        must = [{'multi_match': {'query': this.query, 'fields': this.searchFields}}]
      }

      // Set query
      if (this.category.length > 0) {
        query = {'must': must, 'filter': {'terms': {'category': this.category}}}
      } else {
        query = {'must': must}
      }

      client.search({
        index: this.index,
        type: this.type,
        from: (this.pageNum - 1) * this.perPage,
        size: this.perPage,
        body: {
          'query': {
            'bool': query
          },
          'aggs': {
            'group_by_category': {
              'terms': {
                'field': 'category'
              }
            }
          },
          'sort': sorts,
          'highlight': {
            'pre_tags': ['<strong>'],
            'post_tags': ['</strong>'],
            'fields': {
              'title': {
                'type': 'plain'
              },
              'desc': {
                'type': 'plain'
              }
            }
          }
        }
      }).then(body => {
        this.items = []
        this.buckets = []
        const hits = body.hits.hits
        console.log(hits)
        this.totalCount = body.hits.total
        this.pageCount = Math.ceil(this.totalCount / this.perPage)
        hits.forEach(hit => {
          this.items.push(hit)
        })
        body.aggregations.group_by_category.buckets.forEach(bucket => {
          this.buckets.push(bucket)
        })
      }, function (error) {
        console.log(error)
      })
    },
    goPage: function (pageNum) {
      this.pageNum = pageNum
      this.search()
    },
    apply: function () {
      this.pageNum = 1
      this.search()
    }
  }
}
</script>


<style scoped>
.hpanel>.panel-heading {
    color: inherit;
    font-weight: 600;
    padding: 10px 4px;
    transition: all .3s;
    border: 1px solid transparent;
}
.hpanel .hbuilt.panel-heading {
    border-bottom: 0;
}
.hpanel>.panel-footer, .hpanel>.panel-section {
    color: inherit;
    border: 1px solid #eaeaea;
    border-top: 0;
    font-size: 90%;
    background: #f7f9fa;
    padding: 10px 15px;
}
.hpanel.panel-collapse>.panel-heading, .hpanel .hbuilt {
    background: #fff;
    border-color: #eaeaea;
    border: 1px solid #eaeaea;
    padding: 10px 10px;
    border-radius: 2px;
}
.hpanel .panel-body {
    background: #fff;
    border: 1px solid #eaeaea;
    border-radius: 2px;
    padding: 20px;
    position: relative;
}
.hpanel.panel-group .panel-body:first-child {
    border-top: 1px solid #eaeaea;
}
.hpanel.panel-group .panel-body {
    border-top: 0;
}
.panel-collapse .panel-body {
    border: 0;
}
.hpanel {
    background-color: none;
    border: 0;
    box-shadow: none;
    margin-bottom: 25px;
}
.panel-tools {
    display: inline-block;
    float: right;
    margin-top: 0;
    padding: 0;
    position: relative;
}
.hpanel .alert {
    margin-bottom: 0;
    border-radius: 0;
    border: 1px solid #eaeaea;
    border-bottom: 0;
}
.panel-tools a {
    margin-left: 5px;
    color: #9d9fa2;
    cursor: pointer;
}
.hpanel.hgreen .panel-body {
    border-top: 2px solid #62cb31;
}
.hpanel.hblue .panel-body {
    border-top: 2px solid #3498db;
}
.hpanel.hyellow .panel-body {
    border-top: 2px solid #ffb606;
}
.hpanel.hviolet .panel-body {
    border-top: 2px solid #9b59b6;
}
.hpanel.horange .panel-body {
    border-top: 2px solid #e67e22;
}
.hpanel.hred .panel-body {
    border-top: 2px solid #e74c3c;
}
.hpanel.hreddeep .panel-body {
    border-top: 2px solid #c0392b;
}
.hpanel.hnavyblue .panel-body {
    border-top: 2px solid #34495e;
}
.hpanel.hbggreen .panel-body {
    background: #62cb31;
    color: #fff;
    border: 0;
}
.hpanel.hbgblue .panel-body {
    background: #3498db;
    color: #fff;
    border: 0;
}
.hpanel.hbgyellow .panel-body {
    background: #ffb606;
    color: #fff;
    border: 0;
}
.hpanel.hbgviolet .panel-body {
    background: #9b59b6;
    color: #fff;
    border: 0;
}
.hpanel.hbgorange .panel-body {
    background: #e67e22;
    color: #fff;
    border: 0;
}
.hpanel.hbgred .panel-body {
    background: #e74c3c;
    color: #fff;
    border: 0;
}
.hpanel.hbgreddeep .panel-body {
    background: #c0392b;
    color: #fff;
    border: 0;
}
.hpanel.hbgnavyblue .panel-body {
    background: #34495e;
    color: #fff;
    border: 0;
}
</style>
