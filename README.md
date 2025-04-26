# GUVI-Hackathon
import streamlit as st
import os
import openai
import json
from dotenv import load_dotenv

# Load environment variables
load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")

# Initialize session 
if "chat_history" not in st.session_state:
    st.session_state.chat_history = []

if "question_index" not in st.session_state:
    st.session_state.question_index = 0

if "start" not in st.session_state:
    st.session_state.start = False

if "summary" not in st.session_state:
    st.session_state.summary = []

# Predefined sample questions (will be generated via LLM)
sample_questions = {
    "Technical": [
        "Explain how a hash table works.",
        "Write a function to reverse a linked list.",
        "How would you design a URL shortening service like bit.ly?"
    ],
    "Behavioral": [
        "Tell me about a time you had a conflict at work.",
        "Describe a situation where you showed leadership.",
        "How do you handle tight deadlines?"
    ]
}

# Evaluation Function (LLM call)
def evaluate_answer(question, answer, mode):
    prompt = f"""
You are an interview coach. Evaluate the following response.
Question: {question}
Answer: {answer}
Mode: {mode}

For Technical: score based on correctness, clarity, efficiency, and completeness.
For Behavioral: score based on STAR structure, clarity, and relevance.

Provide feedback and a score out of 10.
"""
    
    response = openai.ChatCompletion.create(
        model="gpt-4-turbo",
        messages=[
            {"role": "system", "content": "You are an expert interview coach."},
            {"role": "user", "content": prompt}
        ]
    )
    return response.choices[0].message.content.strip()

# Streamlit UI
st.title("Interview Preparation Bot")

# Role & Domain
role = st.selectbox("Choose a Job Role:", ["Software Engineer", "Product Manager", "Data Analyst"])
domain = st.text_input("Optional: Specify a domain (e.g., backend, ML, etc.)")

# Mode
mode = st.radio("Choose Interview Mode:", ["Technical", "Behavioral"])

# Start interview
if st.button("Start Interview"):
    st.session_state.start = True
    st.session_state.question_index = 0
    st.session_state.chat_history = []
    st.session_state.summary = []

if st.session_state.start:
    index = st.session_state.question_index
    questions = sample_questions[mode]
    if index < len(questions):
        question = questions[index]
        st.text_area("Question:", value=question, height=100, disabled=True)
        user_answer = st.text_area("Your Answer:", key=f"answer_{index}")

        col1, col2 = st.columns(2)
        with col1:
            if st.button("Submit Answer"):
                feedback = evaluate_answer(question, user_answer, mode)
                st.session_state.chat_history.append((question, user_answer, feedback))
                st.session_state.summary.append({"question": question, "answer": user_answer, "feedback": feedback})
                st.session_state.question_index += 1
                st.rerun()

        with col2:
            if st.button("Skip"):
                st.session_state.question_index += 1
                st.rerun()
    else:
        st.subheader("Interview Complete")
        for entry in st.session_state.summary:
            st.markdown(f"Q: {entry['question']}")
            st.markdown(f"Your Answer: {entry['answer']}")
            st.markdown(f"Feedback: {entry['feedback']}")
            st.markdown("---")

        # for exporting feedback as pdf
        if st.download_button("Download Summary", json.dumps(st.session_state.summary, indent=2), file_name="interview_summary.json"):
            st.success("Summary downloaded.")
