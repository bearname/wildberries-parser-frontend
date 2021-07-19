<template>
  <div>
    <p>
      <router-link to="/login">Logout</router-link>
    </p>
<!--    <h1>Hi {{ user.firstName }}!</h1>-->
    <!--    <em v-if="products.loading">Loading users...</em>-->
    <ul v-if="products.length">
      <li v-for="user in products" :key="user.id">
        {{ user.firstName + ' ' + user.lastName }}
      </li>
    </ul>
    <div id="alert" class="hide"></div>
    <form id="searchForm" @submit.prevent="handleSearch">
      <label for="searchInput"
      ><input id="searchInput" type="text" placeholder="Search"
      />
        <button id="submit" class="btn">Поиск</button>
      </label>
    </form>
    <h6 id="searchTitle" class="text-center text-bold hide"></h6>
    <div class="scrolling-wrapper">
      <table id="products"></table>
    </div>
    <button id="newSearch" v-on:click="newSearchHandle" class="btn btn-new-search hide">Новый поиск</button>
    <label for="clipboard" style="opacity: 0"
    ><input id="clipboard" type="text" placeholder="clipboard"
    /></label>
  </div>
</template>

<script>
// import { userService } from '../_services';

import config from "../config";
import {authHeader} from "../_helpers";

export default {
  data() {
    return {
      user: {},
      products: []
    }
  },
  created() {
    this.user = JSON.parse(localStorage.getItem('user'));
    this.products.loading = true;


    // userService.getAll().then(users => this.users = users);
  },
  mounted() {
    let searchFormElement = document.getElementById('searchForm');
    // let searchInputElement = document.getElementById('searchInput');
    let newSearchButton = document.getElementById('newSearch');
    // let productsElement = document.getElementById('products');
    const searchTitleElement = document.getElementById('searchTitle');
    const productsResult = localStorage.getItem('productsResult');
    const allParameterNames = localStorage.getItem('allParameterNames');
    const searchTitle = localStorage.getItem('searchTitle');
    if (searchTitle !== null && productsResult !== null && productsResult !== undefined && allParameterNames !== null && allParameterNames !== undefined) {
      this.hide(searchFormElement);
      this.unHide(newSearchButton);
      this.unHide(searchTitleElement);
      this.setSearchSearchTitleText(searchTitleElement, searchTitle);
      this.drawSearchResults(JSON.parse(productsResult), allParameterNames.split(','));
    }
  },
  methods: {
    async searchProductsOnWildberries(searchString) {
      let productsElement = document.getElementById('products');
      try {
        localStorage.removeItem('searchTitle')
        localStorage.removeItem('allParameterNames');
        localStorage.removeItem('productsResult');

        productsElement.innerHTML = "Идет поиск...";
        let response = await this.doGet(`/api/search?search=${searchString}`);
        // console.log(response.data);
        const searchTitleElement = document.getElementById('searchTitle');
        this.unHide(searchTitleElement);
        searchTitleElement.innerText = `Результаты поиск «${this.capitalizeFirstLetter(searchString)}»`;
        let products = response.data;

        this.drawSearchResults(products, response.paramNames);
        localStorage.setItem('searchTitle', searchString)
        localStorage.setItem('allParameterNames', response.paramNames);
        localStorage.setItem('productsResult', JSON.stringify(products));
        return true;
      } catch (err) {
        productsElement.innerHTML = "Ничего не найдено...";
        console.log(err);
        return false;
      }
    },
    async handleSearch(e) {
      e.preventDefault();
      const searchFormElement = document.getElementById('searchForm');
      const searchInputElement = document.getElementById('searchInput');
      const newSearchButton = document.getElementById('newSearch');
      const productsElement = document.getElementById('products');
      productsElement.innerHTML = "";
      this.hide(searchFormElement);
      let searchString = searchInputElement.value;
      await this.searchProductsOnWildberries(searchString);
      this.unHide(newSearchButton);
    },
    newSearchHandle(e) {
      e.preventDefault();
      const searchFormElement = document.getElementById('searchForm');
      const searchInputElement = document.getElementById('searchInput');
      const newSearchButton = document.getElementById('newSearch');
      const productsElement = document.getElementById('products');
      const searchTitleElement = document.getElementById('searchTitle');
      this.unHide(searchFormElement);
      searchInputElement.value = '';
      this.hide(searchTitleElement);
      this.hide(newSearchButton);
      productsElement.innerHTML = "";
    },
    buildTableData(products, allParameters) {
      let table = [];
      let keywordMap = new Map();
      products.forEach(product => {
        let row = [];
        row.push(product.name);
        row.push(`https://www.wildberries.ru/catalog/${product.id}/detail.aspx?targetUrl=WR`);
        row.push(product.salePriceU);

        let productParamMap = new Map();

        for (let keywords of product.parameters) {
          productParamMap.set(keywords.name, keywords.value);
          keywords.value.split(';').forEach(keyword => {
            const key = keyword.toLowerCase().trim();
            let keyValue = keywordMap.get(key);
            if (keyValue === undefined) {
              keyValue = 1;
            } else {
              keyValue++;
            }

            keywordMap.set(key, keyValue);
          });
        }

        for (const name of allParameters.keys()) {
          let value = productParamMap.get(name);
          if (value === undefined) {
            row.push('-');
          } else {
            row.push(value);
          }
        }

        table.push(row);
      });
      return {table, keywordMap};
    },
    buildCellHtml(cellValue, i, keywordMap) {
      let innerHTML = '';
      if (cellValue === '-') {
        innerHTML += `<td class="table-cell keyword-empty">${cellValue}</td>`;
      } else if (i === 2) {
        innerHTML += `<td class="table-cell">${cellValue / 100}</td>`;
      } else if (i > 2) {
        innerHTML += `<td class="table-cell">`;
        let keywords = cellValue.split(';');
        let j = 0;
        for (let keyword of keywords) {
          const key = keyword.toLowerCase().trim();
          let count = keywordMap.get(key);
          // console.log('count');
          // console.log(key);
          // console.log(count);
          if (count === undefined || count <= 1 || (count > 1 && keyword.includes(' шт.'))) {
            innerHTML += `<span class="keyword" >${key}</span>`;
          } else if (count > 1) {
            innerHTML += `<span class="keyword keyword-duplicated" >${key}</span>`;
          }
          if (keywords.length > 1 && j < keywords.length) {
            innerHTML += '; ';
          }
          j++;
        }
        innerHTML += `</td>`;
      } else {
        if (i < 1) {
          innerHTML += `<td class="table-cell table-header">${cellValue}</td>`;
        } else {
          innerHTML += `<td class="table-cell"><a href="${cellValue}" target="_blank"><span style="color: #000000;">${cellValue}</span></a></td>`;
        }
      }
      return innerHTML;
    },
    appendChildElement(parentElement, innerHTML, tagName = 'tr') {
      let insertableElement = document.createElement(tagName);
      insertableElement.innerHTML = innerHTML;
      parentElement.appendChild(insertableElement);
    },
    drawTable(table, keywordMap, productsElement) {

      table.forEach(row => {
        let innerHTML = ``;
        let i = 0;
        for (let cellValue of row) {
          innerHTML += this.buildCellHtml(cellValue, i, keywordMap);
          i++;
        }
        table.push(row);
        this.appendChildElement(productsElement, innerHTML);
      });
    },
    drawSearchResults(products, allParameterNames) {
      const productsElement = document.getElementById('products');
      productsElement.innerHTML = "";

      const allParameters = this.drawHeader(allParameterNames, productsElement);
      const {table, keywordMap} = this.buildTableData(products, allParameters);
      this.drawTable(table, keywordMap, productsElement);
    },
    drawHeader(allParameterNames, productsElement) {
      let html = '<th class="table-cell table-header"></th><th class="table-cell table-header">URL</th><th class="table-cell table-header">Цена</th>';
      let i = 0;
      let allParameters = new Map();

      for (let name of allParameterNames) {
        html += `<th class="table-cell table-header">${name.toUpperCase()}</th>`;
        allParameters.set(name, i);
        i++;
      }
      this.appendChildElement(productsElement, html);

      return allParameters;
    },
    capitalizeFirstLetter(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    setSearchSearchTitleText(searchTitleElement, searchTitle) {
      searchTitleElement.innerText = `Результаты поиск «${this.capitalizeFirstLetter(searchTitle)}»`;
    },
    unHide(element) {
      element.classList.remove('hide');
    },
    hide(element) {
      element.classList.add('hide');
    },
    async doGet(path) {
      let input = `${config.BASE_URL}${path}`;
      console.log(input);
      const response = await fetch(input, {
        headers: authHeader(),
      });
      return await response.json();
    }

  },
};
</script>

<style>
body {
  padding: 50px;
  font: 14px "Lucida Grande", Helvetica, Arial, sans-serif;
}

a {
  color: #00b7ff;
}

.hide {
  visibility: hidden;
  opacity: 0;
}

.image {
  width: 300px;
}

.disabled {
  color: gray;
}

.button:hover {
  cursor: pointer;
}

.scrolling-wrapper {
  display: block;
  overflow-x: auto;
  -ms-overflow-x: auto;
  overflow-y: auto;
  -ms-overflow-y: auto;
}

.table-header {
  background-color: #5d5fe7;
  color: #ffffff;
}

.keyword-duplicated {
  color: #e27a2f;
}

.keyword-empty {
  background-color: #d5d5d5;
  color: #282828;
}

.table-cell {
  max-width: 199px;
  width: 100%;
  border: 1px solid #a6a6a6;
  padding: 4px;
  text-align: center;
  vertical-align: middle;
  overflow-wrap: break-word;
  font-size: 12px;
  text-transform: capitalize;
}

.btn {
  display: inline-block;
  vertical-align: middle;
  margin: 0 0 1rem 0;
  padding: .85em 1em;
  border: 1px solid transparent;
  -webkit-transition: background-color .25s ease-out, color .25s ease-out;
  transition: background-color .25s ease-out, color .25s ease-out;
  font-family: inherit;
  font-size: .9rem;
  -webkit-appearance: none;
  line-height: 1;
  text-align: center;
  cursor: pointer;
  color: #ffffff;
  border-radius: 17px;
  background-color: #5d5fef;
  text-transform: uppercase;
}

.btn-new-search {
  padding: 0.85rem 5rem;
  display: block;
  margin: 80px auto 0;
}
</style>