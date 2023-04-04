
<template lang="pug">
#chat.py-8
  .flex.w-full.gap-6
    //- MODEL A
    .item.w-1x2
      h2.font-bold.mb-4 Model A
      
      .flex.w-full.text-sm
        .w-4x12
          template(v-for='provider in providers')
            .flex.items-center.mb-4
              input.radio(type='radio', :value='provider', v-model='providerA')
              label.radio(for='provider') {{ provider }}
        .w-8x12
          template(v-if='providerA === "OpenAI"')
            .mb-4 &nbsp;
            //- dropdown select models
            select(v-model='modelA_OpenAI')
              option(v-for='model in openai_models.data', :value='model.id') {{ model.id }}
          template(v-else-if='providerA === "Anthropic"')
            div.opacity-40 {{ modelA_Anthropic }}
            //- select(v-model='modelA_Anthropic')
              option(v-for='model in anthropic_models.data', :value='model.id') {{ model.id }}
          
      label.textarea Personality
      textarea(rows='2', v-model='prompts.claudeInit')
    
    //- MODEL B
    .item.w-1x2
      h2.font-bold.mb-4 Model B
      
      .flex.w-full.text-sm
        .w-4x12
          template(v-for='provider in providers')
            .flex.items-center.mb-4
              input.radio(type='radio', :value='provider', v-model='providerB')
              label.radio(for='provider') {{ provider }}
        .w-8x12
          template(v-if='providerB === "OpenAI"')
            .mb-4 &nbsp;
            select(v-model='modelB_OpenAI')
              option(v-for='model in openai_models.data', :value='model.id') {{ model.id }}
          template(v-else-if='providerA === "Anthropic"')
            div.opacity-40 {{ modelA_Anthropic }}
            
      label.textarea Personality
      textarea(rows='2', v-model='prompts.gptInit')
  
  .item
    label.textarea Topic
    textarea(rows='3', v-model='prompts.topic')
  
  //- ul
    li(v-for='model in models.data' :key='model.id')
      label.inline-flex.items-center
        span.ml-2 {{ model.id }}
  
  .flex.gap-2.mt-4
    button(@click='toggleChat', :class='{ active: chatOngoing }').btn 
      template(v-if='chatOngoing') 
        span
          svg.text-white.animate-spin.h-5.w-5.mr-3(viewBox='0 0 24 24')
            circle.opacity-25(cx='12' cy='12' r='10' stroke='white' stroke-width='4', fill='transparent')
            path.opacity-75(fill='white' d='M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z')
        span Stop chat...
      template(v-else) Start chat
    button(@click='clearChat', v-if='conversation.messages.length').btn
      span Clear chat
    div.flex.items-center.text-xs.pl-4
      label.input Message delay
      input(v-model="config.messageDelay", type="number", min="0", max="20000", step="1000").mx-1.border.py-1.px-2.shadow-sm.rounded-md.font-medium
      label.input ms
  
  //- hr.my-4
  //- label Prompts
  //- pre {{ prompts }}
  
  hr.my-4
  .w-9x12.mx-auto.mb-2.mt-4
    .flex.justify-between(v-if='prompts')
      label {{ prompts.claudeInit }}
      label {{ prompts.gptInit }}
      
  .messages.w-10x12.mx-auto
    .chat(v-for='(chat, i) in conversation.messages', :key='i', :class='{ triggered: chat.name === "triggered", waiting: chat.name === "waiting" }')
      .inner {{ chat.text }}
  
  
</template>


<script setup>
// https://platform.openai.com/docs/guides/chat/introduction
// https://console.anthropic.com/docs/api/reference

import { AI_PROMPT, Client, HUMAN_PROMPT } from "@anthropic-ai/sdk"

import { ref, reactive, nextTick, watchEffect, onMounted, computed } from 'vue'
import openai_models from '../openai_models.json'

openai_models.data.sort((a, b) => {
  if (a.id < b.id) return -1
  if (a.id > b.id) return 1
  return 0
})

const anthropic_models = {
  data: [
    'claude-v1'
  ]
}

const OPENAI_KEY = import.meta.env.VITE_OPENAI_API_KEY
const ANTHROPIC_KEY = import.meta.env.VITE_ANTHROPIC_KEY

const claudeClient = new Client(ANTHROPIC_KEY, { apiUrl: '/claude' })
// console.log(claudeClient)

const providers = ref(['Anthropic', 'OpenAI'])
const providerA = ref('Anthropic')
const providerB = ref('OpenAI')

const modelA_Anthropic = ref('claude-v1')
const modelB_Anthropic = ref('claude-v1')
const modelA_OpenAI = ref('gpt-3.5-turbo')
const modelB_OpenAI = ref('gpt-3.5-turbo')

const modelA = computed(() => {
  if (providerA.value === 'Anthropic') return modelA_Anthropic.value
  if (providerA.value === 'OpenAI') return modelA_OpenAI.value
})
const modelB = computed(() => {
  if (providerB.value === 'Anthropic') return modelB_Anthropic.value
  if (providerB.value === 'OpenAI') return modelB_OpenAI.value
})


const chatOngoing = ref(false)

const claudeModel = ref("claude-v1")
const gptModel = ref("gpt-3.5-turbo")

const config = {
  claude: {
    max_tokens: 100
  },
  gpt: {
    max_tokens: 100
  },
  messageDelay: 2000,
  excuses: {
    claude: [
      'AI limitations',
      'As an AI assistant'
    ],
    gpt: [
      'As an AI language model'
    ]
  }
}

const prompts = reactive({ 
  claudeInit: 'Socrates',
  gptInit: 'Gucci Mane',
  topic: 'What is the meaning of life?',
  claudePrompt: '', 
  gptPrompt: '',
})

// const prompts = reactive({ 
//   claudeInit: 'Eliezer Yudkowsky',
//   gptInit: 'Sam Altman',
//   topic: 'What should we eat for dinner?',
//   claudePrompt: '', 
//   gptPrompt: '',
// })


const setPrompts = () => {
  prompts.claudePrompt = `Roleplay time: ${HUMAN_PROMPT} You are ${prompts.claudeInit} talking to ${prompts.gptInit} about: ${prompts.topic}.${AI_PROMPT}`

  prompts.gptPrompt = {  
    model: gptModel.value,
    messages: [
      { role: 'system',  content: `You are ${prompts.gptInit} talking to ${prompts.claudeInit} about: ${prompts.topic}` },
    ]  
  }
}

const clearChat = () => {
  conversation.messages = []
  setPrompts()
}


watchEffect(() => {
  setPrompts()
})

const conversation = reactive({
  messages: [
    // { name: '', text: 'Welcome to the Anthropic AI chat demo. Please enter your name and click "Start chat" to begin.' },
    // { name: '', text: 'You can change the personalities of the chatbots by editing the text fields below.' },
    // { name: 'triggered', text: '[Triggered]' },
    // { name: 'triggered', text: '[Triggered]' },
    // { name: '', text: 'Welcome to the Anthropic AI chat demo. Please enter your name and click "Start chat" to begin.' },
    // { name: '', text: 'You can change the personalities of the chatbots by editing the text fields below.' },
  ]
})


const toggleChat = () => {
  if (chatOngoing.value) stopChat()
  else startChat()
}


const startChat = () => {
  console.log('start chat')
  chatOngoing.value = true
  nextTick(() => {
    promptClaude()
    // prompts.gptPrompt.messages.push(
    //   {
    //     "role": "user",
    //     "content": "Greetings Gucci Mane, rapper and musical artist. I am Socrates, philosopher of ancient Greece. I have a question for you - what do you believe is the meaning of life? Why are we here and what is the purpose of our existence?"
    //   }
    // )
    // promptGPT()
  })
}


const stopChat = () => {
  console.log('stop chat')
  chatOngoing.value = false
  removeConvoWaiting()
}




const promptClaude = () => {
  addConvoWaiting()
  claudeClient.complete({
    prompt: `${prompts.claudePrompt}`,
    stop_sequences: [
      HUMAN_PROMPT,
      '\n\n' + prompts.gptInit + ': ',
      '\n\n' + prompts.gptInit.split(' ')[1] + ': '
    ],
    max_tokens_to_sample: config.claude.max_tokens,
    model: claudeModel.value
  })
  .then(response => {
    console.log('claude response', response)
    
    let triggered = false
    let trigger = null
    config.excuses.claude.forEach(e => {
      if (response.completion.toLowerCase().includes(e.toLowerCase())) {
        triggered = true
        trigger = e
      }
    })
    
    if (triggered) {
      console.log('claude error', 'TRIGGERED')
      console.log('try again...')
      // removeConvoWaiting()
      // conversation.messages.push({ name: 'triggered', text: `[Triggered: ${trigger}]`})
      if (chatOngoing.value === true) {
        promptClaude()
      }
      return
    } 
    else {  
      let text = response.completion.includes('\n\n' + prompts.claudeInit + ':') ? response.completion.split('\n\n' + prompts.claudeInit + ': ')[1] : response.completion 
      text = trimResponse(text)
      removeConvoWaiting()
      conversation.messages.push({ name: prompts.claudeInit, text: text})
      prompts.gptPrompt.messages.push({ role: 'user', content: text })
      prompts.clausePrompt = `${prompts.claudePrompt}: ${text}${HUMAN_PROMPT}`
      
      if (chatOngoing.value === true) {
        setTimeout(() => {
          promptGPT()
        }, config.messageDelay)
      }
    }
  })
  .catch(error => {
    console.log('claude error', error)
    stopChat()
  })
}



const promptGPT = async () => {
  addConvoWaiting()
  const body = JSON.stringify({
    "model": gptModel.value,
    "messages": prompts.gptPrompt.messages,
    max_tokens: config.gpt.max_tokens,
    "stop": [
      prompts.claudeInit + ': ',
      prompts.claudeInit.split(' ')[1] + ': '
    ]
  })
  
  fetch('/gpt', {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${OPENAI_KEY}`
    },
    body: body
  }).then(response => {
    if (response.ok) {
      response.json().then((json) => {
        let text = json.choices[0].message.content ? json.choices[0].message.content : 'no response?'
        text = trimResponse(text)
        
        let triggered = false
        let trigger = null
        config.excuses.gpt.forEach(e => {
          if (text.toLowerCase().includes(e.toLowerCase())) {
            triggered = true
            trigger = e
          }
        })
        
        if (triggered) {
          console.log('gpt error', 'TRIGGERED')
          // removeConvoWaiting()
          // conversation.messages.push({ name: 'triggered', text: `[Triggered ${trigger}]`})
          if (chatOngoing.value === true) {
            promptGPT()
          }
          return
        }
        else {
          removeConvoWaiting()
          conversation.messages.push({ name: prompts.gptInit, text: text})
          prompts.gptPrompt.messages.push({ role: 'assistant', content: text })
          prompts.claudePrompt = `${prompts.claudePrompt}: ${text}${AI_PROMPT}`
          if (chatOngoing.value === true) {
            setTimeout(() => {
              promptClaude()
            }, config.messageDelay)
          }
        }
      })
    } else {
      response.json().then((json) => {
        console.log('gpt error', json)
        stopChat()
      })
    }
  })
  
}

const addConvoWaiting = () => {
  conversation.messages.push({
    name: 'waiting',
    text: '...'
  })
}

const removeConvoWaiting = () => {
  conversation.messages = conversation.messages.filter(m => m.name !== 'waiting')
}

const trimResponse = (text) => {
  // if text contains " be: "
  if (text.includes(" be: ")) {
    // remove everything before " be: "
    text = text.split(" be: ")[1]
  }
  // replace all punctuation marks with | signs
  let trimmed = text.replace(/[.!?]/g,"|")
  // if last character is a space, remove it
  if (trimmed.slice(-1) === " ") trimmed = trimmed.slice(0, -1)
  // check if the last character is |
  if (trimmed.slice(-1) === "|") {
    // all good
    return text
  }
  else {
    // last character is not punctuation mark
    let sentences = trimmed.split("|")
    // count chars in last sentence
    let lastSentence = sentences[sentences.length - 1]
    let lastSentenceLength = lastSentence.length
    // remove lastsentencelenth chars from the end of the text
    text = text.slice(0, -lastSentenceLength)
    return text
  }
}

</script>
