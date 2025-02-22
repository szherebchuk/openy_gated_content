<template>
  <Modal
    class="modal-chat text-black"
    :style="{'display': isShowLiveChatModal ? 'table' : 'none'}"
    @close="toggleShowLiveChatModal"
  >
    <template #header>
      <div class="header-info">
        <ChatUserPreferences v-on:click.native="toggleShowLiveChatConfigNameModal"/>
        <span>Chat</span>
        <span class="indicator"
              :class="{online: ratchetServerConnected && !isDisabledLivechat,
                       offline: !ratchetServerConnected || isDisabledLivechat}">
        {{ ratchetServerConnected && !isDisabledLivechat ?
          onlineClientCount + ' people online' : 'offline' }}
      </span>
      </div>
      <button
        v-if="roleIsInstructor && ratchetServerConnected && !isDisabledLivechat"
        class="disable-chat-button"
        @click="disableChat()"
      >
        Disable Chat
      </button>
    </template>
    <template #body>
      <div
        v-for="(msg, index) in liveChatSession"
        :key="index"
        class="message"
        :class="{'d-right': msg.uid === liveChatUserId, 'd-left': msg.uid !== liveChatUserId}"
      >
        <div class="user-icon">
          <span>{{ getMsgAuthor(msg.uid, msg.author, true) }}</span>
        </div>
        <div class="message-card">
          <div class="message-header">
            <h4 class="message-author mb-0">{{ getMsgAuthor(msg.uid, msg.author) }}</h4>
            <div class="message-time">{{ formatDate(msg.date) }}</div>
          </div>
          <div class="message-body">{{ msg.message }}</div>
        </div>
      </div>
      <div v-if="roleIsInstructor && isDisabledLivechat">
        Chat was disabled by you. You can enabled it again on this stream page (right of chat icon)
      </div>
      <div v-else-if="isDisabledLivechat">
        Instructor disabled chat for users
      </div>
      <div v-else-if="!ratchetServerConnected">
        Sorry, chat is not working at current moment
      </div>
    </template>
    <template #footer>
      <input
        type="text"
        placeholder="Message"
        v-model.trim="newMessage"
        :disabled="!ratchetServerConnected || isDisabledLivechat"
        @keyup.enter="messageEnterEvent(newMessage)"
      />
      <button
        @click="messageEnterEvent(newMessage)"
        :disabled="newMessage.length === 0"
      >
        <SendIcon :color="'white'"></SendIcon>
      </button>
    </template>
  </Modal>
</template>

<script>
import { mapGetters, mapActions } from 'vuex';
import Modal from '@/components/modal/Modal.vue';
import SendIcon from '@/components/svg/SendIcon.vue';
import ChatUserPreferences from '@/components/svg/ChatUserPreferences.vue';

export default {
  components: { ChatUserPreferences, Modal, SendIcon },
  data() {
    return {
      newMessage: '',
      chatBody: null,
    };
  },
  computed: {
    ...mapGetters([
      'isShowLiveChatModal',
      'liveChatSession',
      'liveChatUserId',
      'getAppSettings',
      'ratchetServerConnected',
      'bottomScrollOn',
      'onlineClientCount',
      'roleIsInstructor',
      'isDisabledLivechat',
    ]),
  },
  watch: {
    liveChatSession: {
      handler() {
        if (this.bottomScrollOn) {
          this.$nextTick(() => {
            this.scrollDown();
          });
        }
      },
      deep: true,
    },
  },
  methods: {
    ...mapActions([
      'toggleShowLiveChatModal',
      'sendLiveChatMessage',
      'sendLiveChatTechMessage',
      'initRatchetServer',
      'updateBottomScrollOn',
      'toggleShowLiveChatConfigNameModal',
    ]),
    messageEnterEvent(message) {
      if (message.length) {
        this.newMessage = '';
        this.sendLiveChatMessage(message);
      }
    },
    disableChat() {
      const message = {
        disableChat: 1,
      };
      this.sendLiveChatTechMessage(message);
    },
    formatDate(date) {
      return this.$dayjs.date(date).format('ddd, MMM D, YYYY @ h:mm a');
    },
    getMsgAuthor(uid, author, short = false) {
      if (short) {
        return author.slice(0, 1);
      }
      return uid === this.liveChatUserId ? 'Me' : author;
    },
    handleScroll() {
      const scrollY = this.chatBody.scrollTop;

      if ((scrollY + this.chatBody.clientHeight) < this.chatBody.scrollHeight) {
        this.updateBottomScrollOn(false);
      } else {
        this.updateBottomScrollOn(true);
      }
    },
    scrollDown() {
      this.chatBody.scrollTop = this.chatBody.scrollHeight;
    },
  },
  destroyed() {
    this.chatBody.removeEventListener('scroll', this.handleScroll);
  },
  mounted() {
    this.chatBody = this.$el.querySelector('.modal-body');
    this.chatBody.addEventListener('scroll', this.handleScroll);

    if (!this.ratchetServerConnected) {
      this.initRatchetServer();
    }
  },
  updated() {
    // eslint-disable-next-line func-names
    this.$nextTick(() => {
      this.scrollDown();
    });
  },
};
</script>
