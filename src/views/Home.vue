<template>
  <div class="home pa-10">
    <v-app-bar dark color="primary" app>
      <v-toolbar-title>Express, MySQL Pattern Generator</v-toolbar-title>

      <v-spacer></v-spacer>
      <v-btn href="https://github.com/AhmaDev/" target="_BLANK">
        <v-icon left> mdi-github </v-icon>
        ahmadev
      </v-btn>
    </v-app-bar>
    <br /><br />
    <v-row>
      <v-col cols="12" md="3">
        <h2>Table name</h2>
        <v-text-field
          hint="Small case, no spaces, numbers or special characters"
          outlined
          dense
          v-model="name"
        ></v-text-field>
        <h2>Array of table columns</h2>
        <prism-editor
          style="height: 100px"
          class="my-editor"
          :highlight="highlighter"
          v-model="json"
        ></prism-editor>
        <v-btn color="success" @click="generate()" block>
          <v-icon left>mdi-play</v-icon>
          run
        </v-btn>
        <br />
        <v-divider></v-divider>
        <br />
        <v-alert
          icon="mdi-information-outline"
          color="warning"
          colored-border
          border="left"
          dark
          dense
        >
          First item in array should be the unique ID for the table.
        </v-alert>
        <br />
        <v-divider></v-divider>
        <v-checkbox
          label="Mark last item as Timestamp column"
          v-model="lastIsTimestamp"
          dark
          color="primary"
        ></v-checkbox>
        <v-alert
          colored-border
          border="left"
          icon="mdi-check"
          color="success"
          dark
          dense
          >if true, last object will be removed from model</v-alert
        >
      </v-col>
      <v-divider vertical></v-divider>
      <v-col v-if="columns.length > 0" cols="12" md="2">
        <h2>Columns</h2>
        <ul>
          <li v-for="(column, index) in columns" :key="column">
            {{ column }}
            <code v-if="index == 0"> (id)</code>
            <code v-if="index == columns.length - 1 && lastIsTimestamp">
              (Timestamp)</code
            >
          </li>
        </ul>
      </v-col>
      <v-col>
        <v-alert>
          <v-row>
            <v-col class="grow">{{ name }}.models.js</v-col>
            <v-spacer></v-spacer>
            <v-col class="shrink">
              <v-btn small icon @click="copyToClipboard(models)">
                <v-icon>mdi-clipboard-outline</v-icon>
              </v-btn>
            </v-col>
          </v-row>
        </v-alert>
        <prism-editor
          class="my-editor"
          v-model="models"
          :highlight="highlighter"
          line-numbers
          readonly
        ></prism-editor>
        <br />
        <v-alert>
          <v-row>
            <v-col class="grow">{{ name }}.controllers.js</v-col>
            <v-spacer></v-spacer>
            <v-col class="shrink">
              <v-btn small icon @click="copyToClipboard(controllers)">
                <v-icon>mdi-clipboard-outline</v-icon>
              </v-btn>
            </v-col>
          </v-row>
        </v-alert>
        <prism-editor
          class="my-editor"
          v-model="controllers"
          :highlight="highlighter"
          line-numbers
          readonly
        ></prism-editor>
        <br />
        <v-alert>
          <v-row>
            <v-col class="grow">{{ name }}.routes.js</v-col>
            <v-spacer></v-spacer>
            <v-col class="shrink">
              <v-btn small icon @click="copyToClipboard(routes)">
                <v-icon>mdi-clipboard-outline</v-icon>
              </v-btn>
            </v-col>
          </v-row>
        </v-alert>
        <prism-editor
          class="my-editor"
          v-model="routes"
          :highlight="highlighter"
          line-numbers
          readonly
        ></prism-editor>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import { PrismEditor } from "vue-prism-editor";
import "vue-prism-editor/dist/prismeditor.min.css"; // import the styles somewhere

// import highlighting library (you can use any library you want just return html string)
import { highlight, languages } from "prismjs/components/prism-core";
import "prismjs/components/prism-clike";
import "prismjs/components/prism-javascript";
import "prismjs/themes/prism-tomorrow.css";

export default {
  name: "Home",
  components: {
    PrismEditor,
  },
  data: () => ({
    name: "test",
    json: `["idUser", "userName", "password", "createdAt"]`,
    columns: [],
    models: "",
    controllers: "",
    routes: "",
    lastIsTimestamp: false,
  }),
  created() {
    this.generate();
  },
  methods: {
    generate() {
      this.columns = JSON.parse(this.json);
      this.generateModels();
      this.generateController();
      this.generateRoutes();
    },

    generateModels() {
      let nameCap = this.capitalizeFirstLetter(this.name);
      let string = "";
      let length = this.lastIsTimestamp
        ? this.columns.length - 1
        : this.columns.length;
      string += "const connection = require('../helpers/db');";
      string += "\n\n";
      string += `const ${nameCap} = function (${this.name}) {\n`;
      for (let i = 1; i < length; i++) {
        string += `    this.${this.columns[i]} = ${this.name}.${this.columns[i]};\n`;
      }
      string += "};";
      string += "\n\n";
      string += `${nameCap}.create = (new${nameCap}, result) => {\n`;
      string += `    connection.query(\`INSERT INTO ${this.name} SET ?\`, new${nameCap}, (err, res) => {\n`;
      string += `        if (err) {\n`;
      string += `            console.log("Add ${this.name} error:", err);\n`;
      string += `            result(err, null);\n`;
      string += `            return;\n`;
      string += `        }\n`;
      string += `        result(null, { ${this.columns[0]}: res.insertId, ...new${nameCap} })\n`;
      string += `    })\n`;
      string += `};\n`;
      string += "\n\n";
      string += `${nameCap}.getAll = (result) => {
    connection.query(\`SELECT * FROM ${this.name}\`, (err, res) => {
        result(null, res);
    });
};`;
      string += "\n\n";
      string += `${nameCap}.findById = (id, result) => {\n`;
      string += `    connection.query(\`SELECT * FROM ${this.name} WHERE ${this.columns[0]} = \${id}\`, (err, res) => {\n`;
      string += `        if (err) {\n`;
      string += `            console.log("Find By ID: ${this.name} error:", err);\n`;
      string += `            result(err, null);\n`;
      string += `            return;\n`;
      string += `        }\n`;
      string += `        if (res.length == 0) {
        result({ kind: "not_found" }, null);
        } else {
          result(null, res[0]);
        }
    })
};\n`;
      string += "\n\n";
      string += `${nameCap}.update = (id, ${this.name}, result) => {
        connection.query(\`UPDATE ${this.name} SET ? WHERE ${this.columns[0]} = \${id}\`, ${this.name}, (err, res) => {
          if (err) {
            console.log("Error while editing a ${this.name}", err);
            result(err, null);
            return;
        }
        result(null, { ${this.columns[0]}: id, ...${this.name} });
    })
}\n`;
      string += `${nameCap}.delete = (id, result) => {
    connection.query(\`DELETE FROM ${this.name} WHERE ${this.columns[0]} = \${id}\`, (err, res) => {
        if (err) {
            console.log("Error while deleting a ${this.name}", err);
            result(err, null);
            return;
        }
        result(null, { message: \`${nameCap} ID \${id} has been deleted successfully\` });
    });
}
module.exports = ${nameCap};\n`;
      this.models = string;
    },
    generateController() {
      let nameCap = this.capitalizeFirstLetter(this.name);
      let length = this.lastIsTimestamp
        ? this.columns.length - 1
        : this.columns.length;
      let string = "";
      string += `const ${nameCap} = require('../models/${this.name}.model');\n`;
      string += `exports.create = (req, res) => {
    if (!req.body) {
        res.status(400).send({
            message: "Body is empty"
        });
    }
    const ${this.name} = new ${nameCap}({\n`;
      for (let i = 1; i < length; i++) {
        string += `        ${this.columns[i]}: req.body.${this.columns[i]},\n`;
      }
      string += `    });\n`;
      string += `    ${nameCap}.create(${this.name}, (err, data) => {
        if (err) res.sendStatus(500);
        else res.send(data);
    });\n`;
      string += `};\n`;
      string += `exports.findAll = (req, res) => {
    ${nameCap}.getAll((err, data) => {
        if (err) res.sendStatus(500);
        else res.send(data);
    });
};

exports.findOne = (req, res) => {
    ${nameCap}.findById(req.params.id, (err, data) => {
        if (err) {
            if (err.kind == "not_found") res.sendStatus(404);
            else res.sendStatus(500);
        }
        else res.send(data);
    })
};\n`;

      string += `exports.updateOne = (req, res) => {
    ${nameCap}.update(req.params.id, req.body, (err, data) => {
        if (err) res.sendStatus(500);
        else res.send(data);
    })
}

exports.deleteOne = (req, res) => {
    ${nameCap}.delete(req.params.id, (err, data) => {
        if (err) res.sendStatus(500);
        else res.send(data);
    })
}`;
      this.controllers = string;
    },
    generateRoutes() {
      let nameCap = this.capitalizeFirstLetter(this.name);
      let string = "";
      string += `var express = require('express');
var router = express.Router();
const ${this.name} = require('../controllers/${this.name}.controller');\n\n`;
      string += `router.get('/${this.name}s', ${this.name}.findAll);
router.post('/add${nameCap}', ${this.name}.create);
router.get('/${this.name}/:id', ${this.name}.findOne);
router.put('/${this.name}/:id', ${this.name}.updateOne);
router.delete('/${this.name}/:id', ${this.name}.deleteOne);

module.exports = router;
`;
      this.routes = string;
    },
    capitalizeFirstLetter(string) {
      return string.charAt(0).toUpperCase() + string.slice(1);
    },
    highlighter(code) {
      return highlight(code, languages.js); // languages.<insert language> to return html with markup
    },
    async copyToClipboard(mytext) {
      try {
        await navigator.clipboard.writeText(mytext);
        this.$toast.open({
          type: "info",
          message: "Copied!",
          duration: 3000,
        });
      } catch ($e) {
        this.$toast.open({
          type: "error",
          message: "Error",
          duration: 3000,
        });
      }
    },
  },
};
</script>

<style>
/* required class */
.my-editor {
  /* we dont use `language-` classes anymore so thats why we need to add background and text color manually */
  background: #2d2d2d;
  color: #ccc;
  height: 500px;
  /* you must provide font-family font-size line-height. Example: */
  font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
  font-size: 14px;
  line-height: 1.5;
  padding: 5px;
}

/* optional class for removing the outline */
.prism-editor__textarea:focus {
  outline: none;
}
</style>