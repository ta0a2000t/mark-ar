<template>
  <div
    name=""
    ref="texteditor"
    autofocus="true"
    class="flex-1 w-full code-editor"
    id="editor"
  ></div>
</template>

<script>
import "quill/dist/quill.bubble.css";
import hljs from "highlight.js";

import Quill from "quill";
import QuillMarkdown from "quilljs-markdown";
import { onMounted, ref } from "vue";
import { deltaToMarkdown } from "../lib/quill/delta-md.js";
import { MarkdownToQuill } from "md-to-quill-delta";

// Import Delta for clipboard matching
import Delta from "quill-delta";

export default {
  name: "EditorRich",
  emits: ["change"],
  props: {
    opsState: String,
    initialCode: String,
  },
  setup(props, { emit }) {
    const texteditor = ref(null);
    let quill;
    onMounted(() => {
      quill = new Quill("#editor", {
        theme: "bubble",
        modules: {
          syntax: {
            hljs,
          },
          toolbar: [
            ["bold", "italic", "underline", "strike"],
            ["code-block"],
            // Removed direction and align buttons to prevent user modification
          ],
          clipboard: {
            matchers: [
              // Custom matcher to strip unwanted formatting and apply desired defaults
              [
                Node.ELEMENT_NODE, // This matcher applies to all element nodes in the pasted HTML
                (node, delta) => {
                  let newDelta = new Delta();
                  delta.ops.forEach((op) => {
                    let newAttributes = { ...op.attributes }; // Start with existing attributes from the pasted content

                    // Explicitly remove color and background attributes
                    delete newAttributes.color;
                    delete newAttributes.background;
                    // You might want to remove other inline styling attributes to enforce consistency
                    // e.g., font, size, style, class (if they bring unwanted styling)
                    delete newAttributes.font;
                    delete newAttributes.size;
                    // Be cautious with removing all `style` or `class` attributes,
                    // as some might be necessary for basic layout or semantic meaning.
                    // For a "one to one to original markdown form" with no extra styling,
                    // you'd typically remove most presentation-related attributes.

                    // Apply desired block-level formats to all lines/blocks
                    newAttributes.direction = "rtl";
                    newAttributes.align = "right";

                    // Insert the text content with the filtered and enforced attributes
                    newDelta.insert(
                      op.insert,
                      Object.keys(newAttributes).length > 0
                        ? newAttributes
                        : undefined,
                    );
                  });
                  return newDelta;
                },
              ],
            ],
          },
        },
      });

      // Set default direction and alignment immediately after Quill initialization
      quill.format("direction", "rtl");
      quill.format("align", "right");

      // enable markdown conversion
      new QuillMarkdown(quill, {
        debug: true,
      });

      const converter = new MarkdownToQuill({});
      let ops = [];

      try {
        ops = JSON.parse(props.opsState);
      } catch (err) {
        // Migration change to move from
        // storing markdown to quill delta
        // if a syntax error is found, try converting it
        if (err instanceof SyntaxError) {
          ops = converter.convert(props.opsState);
        }
      }

      quill.setContents(ops);
      // Re-apply formats after setting initial content to ensure consistency
      quill.format("direction", "rtl");
      quill.format("align", "right");

      quill.on("text-change", () => {
        const { ops } = quill.getContents();
        const markdownCode = deltaToMarkdown(ops);
        console.log(markdownCode);
        emit("change", {
          code: markdownCode,
          ops: JSON.stringify(ops),
        });
      });

      if (texteditor.value) {
        const editor = texteditor.value.querySelector(".ql-editor");
        texteditor.value.addEventListener("click", (e) => {
          if (e.target.id != "editor") {
            return;
          }
          editor.focus();
        });
        editor.focus();
      }
    });

    return {
      texteditor,
    };
  },
};
</script>

<style scoped>
/* Any component-specific styles can go here if needed */
</style>
