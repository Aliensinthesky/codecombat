<script>
  import { codemirror } from 'vue-codemirror'

  // TODO dynamically import these
  import 'codemirror/mode/javascript/javascript'
  import 'codemirror/mode/python/python'
  import 'codemirror/lib/codemirror.css'

  import BaseInteractiveTitle from '../common/BaseInteractiveTitle'
  const toUriFilePath = asset => encodeURI(`/file/${asset}`)
  export default {
    components: {
      codemirror,
      'base-interactive-title': BaseInteractiveTitle
    },

    props: {
      interactive: {
        type: Object,
        required: true,
        default: () => ({})
      },

      localizedInteractiveConfig: {
        type: Object,
        required: true
      },

      interactiveSession: {
        type: Object
      },

      codeLanguage: {
        type: String
      }
    },

    data () {
      const language = (this.codeLanguage || "").toLowerCase() === 'javascript' ? 'javascript' : 'python'
      // selectedAnswer starts with the `lineToReplace` line from SAMPLE_CODE.
      // TODO handle_error_ozaria - this can crash with invalid input.
      const startingLine = this.localizedInteractiveConfig.starterCode.trim().split('\n')[this.localizedInteractiveConfig.lineToReplace-1].trim()
      const defaultImage = toUriFilePath(this.interactive.defaultArtAsset)
      return {
        codemirrorReady: false,

        codemirrorOptions: {
          tabSize: 2,
          mode: `text/${language}`,
          lineNumbers: true,
          readOnly: 'nocursor'
        },

        selectedAnswer: { id: -1, text: startingLine, triggerArt: defaultImage}
      }
    },

    computed: {
      sampleCodeSplit () {

        const splitSampleCode = this.localizedInteractiveConfig.starterCode
          .trim()
          .split('\n')
          .map(line => line.trim())

        return [
          splitSampleCode.slice(0, this.localizedInteractiveConfig.lineToReplace-1).join('\n'),
          splitSampleCode.slice(this.localizedInteractiveConfig.lineToReplace).join('\n')
        ]
      },

      code () {
        const splitSampleCode = this.sampleCodeSplit

        let selectedAnswerLine = ''
        if (this.selectedAnswer) {
          selectedAnswerLine = this.selectedAnswer.text.trim()
        }

        return `${splitSampleCode[0]}\n${selectedAnswerLine}\n${splitSampleCode[1]}`.trim()
      },

      answerOptions () {
        return this.localizedInteractiveConfig.choices
      },

      codemirror () {
        return this.$refs.codeMirrorComponent.codemirror
      }
    },

    watch: {
      selectedAnswer () {
        this.updateHighlightedLine()
      }
    },

    methods: {
      selectAnswer (answer) {
        this.selectedAnswer = {
          ...answer,
          triggerArt: toUriFilePath(answer.triggerArt)
        }
      },

      onCodeMirrorReady () {
        this.codemirrorReady = true
        this.updateHighlightedLine()
      },

      updateHighlightedLine () {
        // FIXME: This method doesn't seem to work at all. Line style is not applied.
        if (!this.codemirrorReady) {
          return
        }
        if (!this.selectedAnswer || this.selectedAnswer.id !== -1) {
          this.codemirror.addLineClass(this.localizedInteractiveConfig.lineToReplace, 'background', 'highlight-line')
        } else {
          this.codemirror.removeLineClass(this.localizedInteractiveConfig.lineToReplace, 'background', 'highlight-line')
        }
      }
    }
  }
</script>

<template>
<div class="insert-code-container">
    <base-interactive-title
      :interactive="interactive"
    />

    <div class="question-container">
      <ul class="question">
        <li
          v-for="answerOption in answerOptions"
          :key="answerOption.id"
        >
          <button @click="selectAnswer(answerOption)">
            {{ answerOption.text }}
          </button>
        </li>
      </ul>

      <div class="answer">
        <codemirror
          ref="codeMirrorComponent"
          :value="code"
          :options="codemirrorOptions"

          @ready="onCodeMirrorReady"
        />
      </div>

      <div class="art-container">
        <img
          :src="this.selectedAnswer.triggerArt"
          alt="Art!"
        >
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
  .insert-code-container {
    display: flex;
    flex-direction: column;
  }
  .insert-code-container .question-container {
    display: flex;
    flex-direction: row;
    ul.question {
      width: 30%;
      display: flex;
      flex-direction: column;
      align-items: center;
      list-style: none;
      margin: 0;
      padding: 0;
      li {
        margin: 0 0 10px;
        padding: 0;
        width: 70%;
        &:last-of-type {
          margin-bottom: 0;
        }
        button {
          width: 100%;
          height: 20px;
          background: transparent;
          border: 1px solid black;
        }
      }
    }
    .answer {
      width: 30%;
      flex-grow: 1;
      /deep/ .highlight-line {
        background-color: #f8ff89;
      }
    }
    .art-container {
      flex-grow: 1;
      display: flex;
      align-items: center;
      justify-content: center;
      padding: 15px;
      img {
        max-width: 100%;
        max-height: 100%;
      }
    }
  }
</style>
