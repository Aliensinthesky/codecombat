<script>
  import { codemirror } from 'vue-codemirror'
  import { putSession } from 'ozaria/site/api/interactive'
  // TODO dynamically import these
  import 'codemirror/mode/javascript/javascript'
  import 'codemirror/mode/python/python'
  import 'codemirror/lib/codemirror.css'

  import BaseInteractiveTitle from '../common/BaseInteractiveTitle'
  import BaseButton from '../common/BaseButton'
  import ModalInteractive from '../common/ModalInteractive.vue'

  const DEFAULT_ID = '-1'

  const toUriFilePath = asset => encodeURI(`/file/${asset}`)
  export default {
    components: {
      codemirror,
      'base-interactive-title': BaseInteractiveTitle,
      'base-button': BaseButton,
      'modal-interactive': ModalInteractive
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
      // TODO handle_error_ozaria - this can crash with invalid input.
      const startingLine = this.localizedInteractiveConfig.starterCode.trim().split('\n')[this.localizedInteractiveConfig.lineToReplace-1].trim()
      const defaultImage = toUriFilePath(this.interactive.defaultArtAsset)

      const defaultSelectedAnswer = { id: DEFAULT_ID, text: startingLine, triggerArt: defaultImage }
      return {
        codemirrorReady: false,

        codemirrorOptions: {
          tabSize: 2,
          mode: `text/${language}`,
          lineNumbers: true,
          readOnly: 'nocursor'
        },

        selectedAnswer: defaultSelectedAnswer,

        displayModal: false
      }
    },

    mounted () {
      this.stateFromCompleteSession()
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
      },

      solution () {
        return {
          correct: this.localizedInteractiveConfig.solution === this.selectedAnswer.choiceId,
          submittedSolution: this.selectedAnswer.choiceId
        }
      }
    },

    watch: {
      selectedAnswer () {
        setTimeout(() => this.updateHighlightedLine(), 10)
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
        if (!this.codemirrorReady) {
          return
        }
        if (this.selectedAnswer.id !== DEFAULT_ID) {
          this.codemirror.addLineClass(this.localizedInteractiveConfig.lineToReplace-1, 'background', 'highlight-line')
        } else {
          this.codemirror.removeLineClass(this.localizedInteractiveConfig.lineToReplace-1, 'background', 'highlight-line')
        }
      },

      submitSolution () {
        this.displayModal = true
      },

      async closeModal () {
        if (this.solution.submittedSolution === DEFAULT_ID) {
          return
        }

        await putSession(this.interactive._id, {
          json: {
            codeLanguage: this.codeLanguage,
            submission: this.solution
          }
        })
        if (this.solution.correct) {
          this.$emit('completed')
        } else {
          // Reset component to base state
          Object.assign(this.$data, this.$options.data.apply(this))
          this.stateFromCompleteSession()
        }
        this.displayModal = false
      },

      stateFromCompleteSession () {
        const correctSubmissionId = ((this.interactiveSession || {}).submissions || []).find(({ correct }) => correct)
        if (!(correctSubmissionId || {}).submittedSolution) {
          return
        }

        const answer = this.localizedInteractiveConfig.choices.find(({ choiceId }) => choiceId === correctSubmissionId.submittedSolution)
        if (!answer) {
          console.warn('Session answer id doesn\'t match given choices. Proceeding without setting session state.')
          return
        }
        this.selectAnswer(answer)
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
        <base-button
          :onClick="submitSolution"
          text="Submit"
        >
        </base-button>
      </div>

      <div class="art-container">
        <img
          :src="this.selectedAnswer.triggerArt"
          alt="Art!"
        >
      </div>
    </div>
    <div v-if="displayModal">
      <modal-interactive
        @close="closeModal"
      >
    <template v-slot:body>
      <h1>{{solution.correct ? "Did it!" : "Try Again!"}}</h1>
    </template>

      </modal-interactive>
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
