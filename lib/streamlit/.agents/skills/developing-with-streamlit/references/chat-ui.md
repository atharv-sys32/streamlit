
# Streamlit chat interfaces

Build conversational UIs with Streamlit's chat elements.

## Basic chat structure

```python
import streamlit as st

if "messages" not in st.session_state:
    st.session_state.messages = []

# Display chat history
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.write(msg["content"])

# Handle new input
if prompt := st.chat_input("Ask a question"):
    st.session_state.messages.append({"role": "user", "content": prompt})

    with st.chat_message("user"):
        st.write(prompt)

    with st.chat_message("assistant"):
        response = get_response(prompt)  # Your LLM call
        st.write(response)

    st.session_state.messages.append({"role": "assistant", "content": response})
```

## Streaming responses

Use `st.write_stream` for token-by-token display. Pass any generator that yields strings, including the OpenAI generator directly:

```python
def get_streaming_response(prompt):
    # Replace with your LLM client (OpenAI, Anthropic, Cortex, etc.)
    for chunk in your_llm_client.stream(prompt):
        yield chunk

with st.chat_message("assistant"):
    response = st.write_stream(get_streaming_response(prompt))

st.session_state.messages.append({"role": "assistant", "content": response})
```

With OpenAI, you can pass the stream directly:

```python
from openai import OpenAI

client = OpenAI()
with st.chat_message("assistant"):
    stream = client.chat.completions.create(
        model="gpt-4o",
        messages=st.session_state.messages,
        stream=True,
    )
    response = st.write_stream(stream)
```

## Agent reasoning / status indicators

Use `st.status` to show the progress of a long-running or multi-step task (tool calls, retrieval, reasoning steps). For agent/reasoning UIs, `type="compact"` gives a minimal, low-chrome inline indicator — a nice-to-have when the default bordered box feels too heavy, though not required. Either way, reach for `st.status` itself: don't hand-roll it with CSS `st.markdown` divs or `st.expander`, and don't settle for a bare `st.write` dump. Put the status where the work runs so its steps appear as soon as the task runs. Don't gate it behind a button that exists only to launch the task — but running it in response to user input (e.g. `if prompt := st.chat_input(...)`) is fine.

```python
import streamlit as st

# GOOD: compact, low-chrome status that renders as soon as the app runs (not
# behind a button). expanded=True shows the steps immediately; omit it or pass
# False for a collapsed toggle.
with st.status("Thinking...", type="compact", expanded=True) as status:
    st.write("Searching for context...")
    status.update(label="Summarising results...")  # relabel mid-task; the with
    st.write("Comparing the top results...")        # block auto-completes on exit

# Also fine: the default st.status renders a bordered box — a real status
# container, just more chrome. type="compact" is a nice-to-have, not required.
with st.status("Thinking..."):
    st.write("Searching for context...")
```

```python
# BAD: faking it with an expander or CSS divs — st.status is the right primitive.
with st.expander("Thinking..."):
    st.write("Searching for context...")
st.markdown("<div class='status'>Thinking...</div>", unsafe_allow_html=True)

# BAD: gating behind a button whose only job is to launch the task — nothing
# renders on load. (Running in response to real input like st.chat_input is fine;
# a bare "Run" button that only kicks off the task is not.)
if st.button("Run agent"):
    with st.status("Working...", type="compact"):
        st.write("Searching for context...")
```

Notes:
- **Render it on load.** Write the step lines (`st.write(...)`) directly inside the `with` block so the status shows as soon as the task runs. Don't wrap it in a button that exists only to launch the task; running it in response to genuine user input (e.g. `if prompt := st.chat_input(...)`) is fine.
- **Stream into it.** `st.write_stream(...)` works inside the `with st.status(...)` block — streaming reasoning steps or tokens into a compact status is the common agent pattern.
- `type` is `"default"` (bordered box) or `"compact"` (minimal inline toggle — a nice-to-have for low-chrome agent UIs, not required). Pass it as a keyword: `type="compact"`.
- `state` must be one of `"running"` (default), `"complete"`, or `"error"` — any other literal raises `StreamlitAPIException`. Same constraint on `status.update(state=...)`.
- The `with` block auto-marks the status `"complete"` on exit; call `status.update(label=..., state=..., expanded=...)` to change it mid-task.
- The compact status renders as a collapsible toggle; use `expanded=True` if you want the steps shown without a click.

## Chat message avatars

Streamlit provides default avatars for "user" and "assistant" roles—only customize if you have a specific need. You can use icons or images:

```python
# With icons
with st.chat_message("assistant", avatar=":material/robot:"):
    st.write(assistant_message)

# With images
with st.chat_message("user", avatar="https://example.com/avatar.png"):
    st.write(user_message)
```

## Suggestion chips

Offer clickable suggestions before the first message. The pills disappear once the user sends a message, creating a clean onboarding experience:

```python
SUGGESTIONS = {
    ":blue[:material/help:] What is Streamlit?": "Explain what Streamlit is",
    ":green[:material/code:] Show me an example": "Show a simple Streamlit example",
}

# Only show before first message - they disappear after
if not st.session_state.messages:
    selected = st.pills("Try asking:", list(SUGGESTIONS.keys()), label_visibility="collapsed")
    if selected:
        # Use the selection as the first prompt
        prompt = SUGGESTIONS[selected]
        st.session_state.messages.append({"role": "user", "content": prompt})
        st.rerun()
```

The `if not st.session_state.messages` check ensures the suggestions only appear on an empty chat. Once a message is added, the pills vanish and the conversation takes over.

## File uploads

Enable file attachments with `accept_file`. When enabled, `st.chat_input` returns a dict-like object with `text` and `files` attributes:

```python
prompt = st.chat_input(
    "Ask about an image",
    accept_file=True,
    file_type=["jpg", "jpeg", "png"],
)

if prompt:
    with st.chat_message("user"):
        if prompt.text:
            st.write(prompt.text)
        if prompt.files:
            st.image(prompt.files[0])

    # Send to vision model
    with st.chat_message("assistant"):
        response = analyze_image(prompt.files[0], prompt.text)
        st.write(response)
```

Use `accept_file="multiple"` to allow multiple files.

## Audio input

Enable voice recording with `accept_audio`. The recorded audio is available as a WAV file:

```python
prompt = st.chat_input("Say something", accept_audio=True)

if prompt:
    if prompt.audio:
        st.audio(prompt.audio)
    if prompt.text:
        st.write(prompt.text)
```

### Dictation with speech-to-text

Convert audio to text and inject it back into the chat input:

```python
prompt = st.chat_input("Say something", accept_audio=True, key="chat")

if prompt and prompt.audio:
    # Transcribe with Whisper or another STT model
    transcript = openai.audio.transcriptions.create(
        model="whisper-1",
        file=prompt.audio,
    )
    # Set the transcribed text as the next input
    st.session_state.chat = transcript.text
    st.rerun()
```

## References

- `snowflake-connection.md` — Database queries and Cortex chat example
- `performance.md` — Caching strategies for LLM calls
- [st.chat_message](https://docs.streamlit.io/develop/api-reference/chat/st.chat_message)
- [st.chat_input](https://docs.streamlit.io/develop/api-reference/chat/st.chat_input)
- [st.write_stream](https://docs.streamlit.io/develop/api-reference/write-magic/st.write_stream)
