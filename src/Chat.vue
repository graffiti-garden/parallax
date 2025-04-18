<script setup lang="ts">
import { computed, ref, type Ref } from "vue";
import {
    useGraffitiSession,
    useGraffitiDiscover,
} from "@graffiti-garden/wrapper-vue";
import {
    chatNameSchema,
    memberUpdateSchema,
    messageSchema,
    type ChatNameObject,
    type MemberUpdateObject,
    type MessageObject,
} from "./schemas";
import { sortByPublished, groupAdjacentBy } from "./utils";
import Membership from "./Membership.vue";
import ChatNameEditor from "./ChatNameEditor.vue";
import SendMessage from "./SendMessage.vue";
import DisplayMessage from "./DisplayMessage.vue";
import DisplayChatNameUpdate from "./DisplayChatNameUpdate.vue";
import DisplayMembershipUpdate from "./DisplayMembershipUpdate.vue";
import type { GraffitiSession } from "@graffiti-garden/api";
import { chatAdmin, parallaxOrProvenance } from "./parallaxOrProvenance";

const props = defineProps<{
    channel: string;
    session: GraffitiSession;
}>();

const graffitiSession = useGraffitiSession();

const admin = chatAdmin(
    () => props.channel,
    () => props.session,
);

const { objects: chatNamesRaw, isInitialPolling: isInitialPollingChatNames } =
    useGraffitiDiscover(
        () => [props.channel],
        () => chatNameSchema(props.channel),
        undefined,
        true,
    );
const chatNamesSorted = sortByPublished<ChatNameObject>(chatNamesRaw);
const chatNames = computed(() =>
    chatNamesSorted.value.filter((chatName) => {
        return (
            parallaxOrProvenance !== "Provenance" ||
            chatName.actor === admin.value
        );
    }),
);

const myChatName = computed(
    () =>
        chatNames.value.filter((c) => c.actor === admin.value).at(0)?.value
            .name,
);

const {
    objects: memberUpdatesRaw,
    isInitialPolling: isInitialPollingMemberUpdates,
} = useGraffitiDiscover(
    () => [props.channel],
    () => memberUpdateSchema(props.channel),
    undefined,
    true,
);
const memberUpdatesAll = sortByPublished<MemberUpdateObject>(memberUpdatesRaw);

const myMembers = computed(() => {
    const members = new Set<string>();
    for (const memberUpdate of memberUpdatesAll.value
        .filter((m) => m.actor === admin.value)
        .toReversed()) {
        if (memberUpdate.value.activity === "Add") {
            members.add(memberUpdate.value.target);
        } else {
            members.delete(memberUpdate.value.target);
        }
    }
    members.add(admin.value);
    return members;
});

// Show all "adds" but only show
// "removes" if the actor is in "myMembers"
const memberUpdates = computed(() => {
    return memberUpdatesAll.value.filter((memberUpdate) => {
        if (
            parallaxOrProvenance === "Provenance" &&
            memberUpdate.actor !== admin.value
        ) {
            return false;
        }
        if (memberUpdate.value.activity === "Add") {
            return true;
        } else {
            return myMembers.value.has(memberUpdate.actor);
        }
    });
});

const { objects: messages_, isInitialPolling: isInitialPollingMessages } =
    useGraffitiDiscover(
        () => [props.channel],
        () => messageSchema(),
        undefined,
        true,
    );
const messages: Ref<MessageObject[]> = computed(() => {
    const messages: MessageObject[] = messages_.value;
    return messages.filter((message) => {
        return myMembers.value.has(message.actor);
    });
});

// Group changes to the same name
const groupedChatNames = groupAdjacentBy(chatNames, (group, chatName) =>
    group.some((c) => c.value.name === chatName.value.name),
);

// Group changes performing the same operation to the same person
const groupedMemberUpdates = groupAdjacentBy(
    memberUpdates,
    (group, memberUpdate) =>
        group.some(
            (m) =>
                m.value.target === memberUpdate.value.target &&
                m.value.activity === memberUpdate.value.activity,
        ),
);

function publishedTime(
    object: ChatNameObject[] | MemberUpdateObject[] | MessageObject,
) {
    if (Array.isArray(object)) {
        return Math.min(...object.map((c) => c.value.published));
    } else {
        return object.value.published;
    }
}

const updates = computed(() => {
    return [
        ...groupedChatNames.value.map(
            (value) => ({ type: "ChatName", value }) as const,
        ),
        ...groupedMemberUpdates.value.map(
            (value) => ({ type: "MemberUpdate", value }) as const,
        ),
        ...messages.value.map((value) => ({ type: "Message", value }) as const),
    ].sort((a, b) => publishedTime(b.value) - publishedTime(a.value));
});

const isMembersOpen = ref(false);
</script>

<template>
    <main>
        <dialog :open="isMembersOpen" @close="isMembersOpen = false">
            <article>
                <header>
                    <h3>Members of "{{ myChatName ?? "Unnamed Chat" }}"</h3>
                    <nav>
                        <button @click="isMembersOpen = false">Close</button>
                    </nav>
                </header>
                <main>
                    <Membership
                        :channel="props.channel"
                        :myMembers="myMembers"
                        :session="session"
                        :admin="admin"
                    />
                </main>
            </article>
        </dialog>
        <article class="chat">
            <header>
                <button @click="$router.push('/')">⇦</button>
                <h2>
                    <ChatNameEditor
                        :channel="props.channel"
                        :myChatName="myChatName"
                        :session="session"
                        :myMembers="myMembers"
                        :admin="admin"
                    />
                </h2>
                <button @click="isMembersOpen = true">Members</button>
            </header>
            <main>
                <ul>
                    <li
                        v-for="update in updates"
                        :key="
                            update.type === 'Message'
                                ? update.value.url
                                : update.value[0].url
                        "
                    >
                        <DisplayMessage
                            v-if="update.type === 'Message'"
                            :message="update.value"
                        />
                        <DisplayChatNameUpdate
                            v-else-if="update.type === 'ChatName'"
                            :group="update.value"
                            :myChatName="myChatName"
                            :myMembers="myMembers"
                            :channel="props.channel"
                            :session="session"
                            :admin="admin"
                        />
                        <DisplayMembershipUpdate
                            v-else-if="update.type === 'MemberUpdate'"
                            :group="update.value"
                            :myMembers="myMembers"
                            :channel="props.channel"
                            :session="session"
                            :admin="admin"
                        />
                    </li>
                </ul>
            </main>

            <footer>
                <SendMessage
                    :channel="channel"
                    :myMembers="myMembers"
                    :session="session"
                />
            </footer>
        </article>
    </main>
</template>

<style>
.chat {
    background: var(--background1);
    display: flex;
    flex-direction: column;
    height: 100vh;

    > main {
        display: contents;
    }

    > footer {
        padding: 1rem;
        border-top: 2px solid var(--background2);
    }
}

header:not(li *) {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 2px solid var(--background2);
    padding: 1rem;
}

button,
input[type="submit"] {
    background: var(--foreground2);
    border: none;
    color: var(--text1);
    border-radius: 1em;
    padding-left: 1em;
    padding-right: 1em;
    padding-top: 0.5em;
    padding-bottom: 0.5em;
    font-size: inherit;
}

:is(button, input[type="submit"]):hover {
    background: var(--foreground1);
    cursor: pointer;
}

.chat ul {
    padding: 1rem;
    flex: 1;
    overflow-y: scroll;
    list-style: none;

    display: flex;
    flex-direction: column-reverse;
    gap: 0.5rem;

    li {
        display: flex;
        flex-direction: column;
        align-items: center;

        h3 {
            margin-bottom: 0.5rem;
            font-size: 1rem;
        }

        aside {
            font-size: 80%;
            display: flex;
            flex-direction: column;
            gap: 0.5em;
            margin: 1rem;
            align-items: center;

            p {
                color: var(--text3);
            }
        }
    }
}

/* Overlay the dialog */
dialog[open] {
    position: fixed;
    top: 50%;
    left: 50%;
    width: 90vw;
    height: 90vh;
    transform: translate(-50%, -50%);
    background: var(--background1);
    border: 3px solid var(--foreground1);
    box-shadow: 0 0 3rem rgba(0, 0, 0, 1); /* shadow */
    color: var(--text1);
    border-radius: 1rem;
    overflow: auto;
    z-index: 1000; /* ensure it floats above page content */

    article {
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    header {
        align-self: stretch;
    }

    main {
        padding: 1rem;
        max-width: 30rem;
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    form {
        input[type="text"] {
            flex: 1;
            border-top-right-radius: 0;
            border-bottom-right-radius: 0;
        }

        input[type="submit"] {
            border-top-left-radius: 0;
            border-bottom-left-radius: 0;
        }
    }
}

input[type="text"] {
    background: var(--foreground2);
    color: var(--text1);
    border: none;
    padding-left: 1rem;
    padding-right: 1rem;
    padding-top: 0.5rem;
    padding-bottom: 0.5rem;
    border-radius: 1rem;
}

input[type="text"]::placeholder {
    color: var(--text4);
}

input[type="text"]:focus {
    outline: none;
}

input[type="submit"] {
    background: var(--highlight);
}

input[type="submit"]:hover {
    background: var(--highlight-hover);
}

form {
    display: flex;
    flex-direction: row;
    justify-content: center;
}
</style>
