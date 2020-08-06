<template>
  <div>
    <loading-dialog :active="deleting" message="Deleting" />

    <confirm-dialog
      :active="confirmDeleteDialog"
      prompt="Delete Note?"
      confirmMessage="delete"
      declineMessage="cancel"
      @confirm="deleteResource"
      @decline="confirmDeleteDialog = false"
    />
    <v-speed-dial
      v-model="fab"
      fixed
      bottom
      right
      direction="top"
      transition="slide-y-reverse-transition"
      class="ma-6"
      v-if="mode === 'read'"
    >
      <template v-slot:activator>
        <v-btn v-model="fab" color="primary" dark fab large>
          <v-icon v-if="fab">mdi-close</v-icon>
          <v-icon v-else>mdi-dots-vertical</v-icon>
        </v-btn>
      </template>
      <v-btn fab dark large color="green" @click="edit">
        <v-icon>mdi-pencil</v-icon>
      </v-btn>
      <v-btn fab dark large color="blue">
        <v-icon>mdi-export</v-icon>
      </v-btn>
      <v-btn fab dark large color="red" @click="confirmDeleteDialog = true">
        <v-icon>mdi-delete</v-icon>
      </v-btn>
    </v-speed-dial>
    <v-btn
      fab
      dark
      fixed
      bottom
      right
      large
      color="success"
      @click="save"
      class="ma-6"
      v-if="mode === 'write'"
    >
      <v-icon large>mdi-check</v-icon>
    </v-btn>
    <v-row justify="center">
      <v-col cols="12" md="10" xl="8">
        <resource-header v-if="mode === 'read'" :resource="resource" />
        <editable-resource-header v-if="mode === 'write'" :resource="resource" />
        <div class="editor">
          <v-card
            :class="(this.$vuetify.theme.dark ? 'markdown-body-dark' : 'markdown-body-light')"
          >
            <v-toolbar v-if="mode === 'write'" color="secondary darken-1">
              <editor-menu-bar :editor="editor" v-slot="{ commands, isActive }">
                <div class="menubar">
                  <v-btn
                    class="menubar__button"
                    :depressed="isActive.bold()"
                    @click="commands.bold"
                  >
                    <v-icon large>mdi-format-bold</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.italic() }"
                    @click="commands.italic"
                  >
                    <v-icon large>mdi-format-italic</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.strike() }"
                    @click="commands.strike"
                  >
                    <v-icon large>mdi-format-strikethrough</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.underline() }"
                    @click="commands.underline"
                  >
                    <v-icon large>mdi-format-underline</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.code() }"
                    @click="commands.code"
                  >
                    <v-icon large>mdi-code-braces</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.paragraph() }"
                    @click="commands.paragraph"
                  >
                    <v-icon large>mdi-format-pilcrow</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.heading({ level: 1 }) }"
                    @click="commands.heading({ level: 1 })"
                  >
                    <v-icon large>mdi-format-header-1</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.heading({ level: 2 }) }"
                    @click="commands.heading({ level: 2 })"
                  >
                    <v-icon large>mdi-format-header-2</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.heading({ level: 3 }) }"
                    @click="commands.heading({ level: 3 })"
                  >
                    <v-icon large>mdi-format-header-3</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.bullet_list() }"
                    @click="commands.bullet_list"
                  >
                    <v-icon large>mdi-format-list-bulleted-square</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.ordered_list() }"
                    @click="commands.ordered_list"
                  >
                    <v-icon large>mdi-format-list-numbered</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.blockquote() }"
                    @click="commands.blockquote"
                  >
                    <v-icon large>mdi-format-quote-close</v-icon>
                  </v-btn>
                  <v-btn
                    class="menubar__button"
                    :class="{ 'is-active': isActive.code_block() }"
                    @click="commands.code_block"
                  >
                    <v-icon large>mdi-code-json</v-icon>
                  </v-btn>
                  <v-btn class="menubar__button" @click="commands.horizontal_rule">
                    <v-icon large>mdi-minus</v-icon>
                  </v-btn>
                  <v-btn class="menubar__button" @click="commands.undo">
                    <v-icon large>mdi-undo</v-icon>
                  </v-btn>
                  <v-btn class="menubar__button" @click="commands.redo">
                    <v-icon large>mdi-redo</v-icon>
                  </v-btn>
                </div>
              </editor-menu-bar>
            </v-toolbar>
            <editor-content class="editor__content" :editor="editor" />
          </v-card>
        </div>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import axios from "axios";
import { Auth } from "aws-amplify";
import { Editor, EditorContent, EditorMenuBar } from "tiptap";
import {
  CodeBlockHighlight,
  Blockquote,
  CodeBlock,
  HardBreak,
  Heading,
  HorizontalRule,
  OrderedList,
  BulletList,
  ListItem,
  TodoItem,
  TodoList,
  Bold,
  Code,
  Italic,
  Link,
  Strike,
  Underline,
  History,
} from "tiptap-extensions";
import javascript from "highlight.js/lib/languages/javascript";
import css from "highlight.js/lib/languages/css";
import python from "highlight.js/lib/languages/python";
import LoadingDialog from "../components/LoadingDialog";
import EditableResourceHeader from "../components/EditableResourceHeader";
import ResourceHeader from "../components/ResourceHeader";
import ConfirmDialog from "../components/ConfirmDialog";

export default {
  components: {
    EditorContent,
    EditorMenuBar,
    ConfirmDialog,
    LoadingDialog,
    EditableResourceHeader,
    ResourceHeader,
  },
  props: ["resource"],
  data() {
    return {
      mode: "read",
      fab: false,
      confirmDeleteDialog: false,
      deleting: false,
      editor: new Editor({
        editable: false,
        extensions: [
          new CodeBlockHighlight({
            languages: {
              javascript,
              css,
              python,
            },
          }),
          new Blockquote(),
          new BulletList(),
          new CodeBlock(),
          new HardBreak(),
          new Heading({ levels: [1, 2, 3] }),
          new HorizontalRule(),
          new ListItem(),
          new OrderedList(),
          new TodoItem(),
          new TodoList(),
          new Link(),
          new Bold(),
          new Code(),
          new Italic(),
          new Strike(),
          new Underline(),
          new History(),
        ],
        content: this.resource.content,
      }),
    };
  },
  watch: {
    mode() {
      var vm = this;
      vm.editor.setOptions({
        editable: vm.mode == "read" ? false : true,
      });
    },
  },
  mounted() {
    var vm = this;
    this._keyListener = function (e) {
      console.log(e)
      if (e.key === "i" && (e.ctrlKey || e.metaKey) && vm.mode === "read") {
        e.preventDefault();
        vm.edit()
      } else if (e.key === "s" && (e.ctrlKey || e.metaKey) && vm.mode === "write") {
        e.preventDefault();
        vm.save()
      } else if (e.key === "h" && (e.ctrlKey || e.metaKey)) {
        e.preventDefault();
        vm.$router.push({ name: "Index" });
      } 
    };
    document.addEventListener("keydown", this._keyListener.bind(this));
  },

  methods: {
    clearSelection: function () {
      if (window.getSelection) {
        window.getSelection().removeAllRanges();
      } else if (document.selection) {
        document.selection.empty();
      }
    },
    edit: function () {
      var vm = this;
      vm.mode = "write";
      vm.fab = false;
    },
    save: function () {
      var vm = this;
      vm.resource.content = vm.editor.getHTML();
      Auth.currentAuthenticatedUser()
        .then((data) => {
          axios({
            method: vm.$variables.api.updateResource.method,
            url:
              vm.$variables.api.updateResource.url +
              vm.$route.params.resource_id,
            headers: {
              Authorization: data.signInUserSession.idToken.jwtToken,
            },
            data: {
              resource: vm.resource,
            },
          })
            .then((response) => {
              console.log(response);
              // correct the updated_at time with new time from server
              vm.resource.updated_at = response.data.resource.updated_at;
            })
            .catch((err) => {
              console.error(err);
            });
        })
        .catch((err) => {
          console.log(err);
        });

      vm.mode = "read";
      vm.clearSelection();
      console.log(vm.resource.content);
    },
    deleteResource: function () {
      var vm = this;
      vm.confirmDeleteDialog = false;
      vm.deleting = true;
      Auth.currentAuthenticatedUser()
        .then((data) => {
          axios({
            method: vm.$variables.api.deleteResource.method,
            url:
              vm.$variables.api.deleteResource.url +
              vm.$route.params.resource_id,
            headers: {
              Authorization: data.signInUserSession.idToken.jwtToken,
            },
          })
            .then((response) => {
              console.log(response);
              vm.$router.push({ name: "Index" });
            })
            .catch((err) => {
              console.error(err);
            });
        })
        .catch((err) => {
          console.log(err);
        });
    },
  },
  beforeDestroy() {
    this.editor.destroy();
    document.removeEventListener("keydown", this._keyListener);
  },
};
</script>

<style lang="scss">
:focus {
  outline: none;
}
.v-btn:not(.v-btn--round).v-size--default {
  padding: 0;
  min-width: 0;
  min-height: 0;
  height: 3em;
  width: 3em;
  margin: 0.25em;
}

.ProseMirror {
  padding-top: 1em;
  padding-bottom: 1em;
  padding-left: 1.5em;
  padding-right: 1.5em;
}
pre {
  &::before {
    content: attr(data-language);
    text-transform: uppercase;
    display: block;
    text-align: right;
    font-weight: bold;
    font-size: 0.6rem;
  }
  code {
    .hljs-comment,
    .hljs-quote {
      color: #999999;
    }
    .hljs-variable,
    .hljs-template-variable,
    .hljs-attribute,
    .hljs-tag,
    .hljs-name,
    .hljs-regexp,
    .hljs-link,
    .hljs-name,
    .hljs-selector-id,
    .hljs-selector-class {
      color: #f2777a;
    }
    .hljs-number,
    .hljs-meta,
    .hljs-built_in,
    .hljs-builtin-name,
    .hljs-literal,
    .hljs-type,
    .hljs-params {
      color: #f99157;
    }
    .hljs-string,
    .hljs-symbol,
    .hljs-bullet {
      color: #99cc99;
    }
    .hljs-title,
    .hljs-section {
      color: #ffcc66;
    }
    .hljs-keyword,
    .hljs-selector-tag {
      color: #6699cc;
    }
    .hljs-emphasis {
      font-style: italic;
    }
    .hljs-strong {
      font-weight: 700;
    }
  }
}
</style>