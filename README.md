import streamlit as st
import openai
from typing import Dict, Tuple
import re
from config import OPENAI_API_KEY
import traceback

class CodeReviewer:
    def __init__(self, model="gpt-3.5-turbo"):
        
        try:
            openai.api_key = OPENAI_API_KEY
            self.model = model
        except Exception as e:
            raise Exception("Failed to initialize OpenAI API key. Check your .env file.")
        
    def test_api_key(self) -> bool:
      
        try:
            openai.ChatCompletion.create(
                model=self.model,
                messages=[{"role": "user", "content": "test"}],
                max_tokens=5
            )
            return True
        except Exception:
            return False
        
    def analyze_code(self, code: str) -> Tuple[Dict, str]:
        
        try:
       
            if not code.strip():
                raise ValueError("Empty code provided")
            
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "You are a senior Python developer doing code review. Provide detailed, actionable feedback focusing on code quality, performance, and best practices."},
                    {"role": "user", "content": prompt}
                ],
                temperature=0.3,
                max_tokens=2000
            )
            
            review_text = response.choices[0].message.content
          
            analysis = {
                "bugs": [],
                "style": [],
                "performance": []
            }
            
            try:
                bugs = re.findall(r"BUGS:\n(.*?)(?=STYLE:)", review_text, re.DOTALL)
                if bugs:
                    analysis["bugs"] = self._parse_list(bugs[0])
            except Exception:
                st.warning("Error parsing bugs section")
                
            try:
                style = re.findall(r"STYLE:\n(.*?)(?=PERFORMANCE:)", review_text, re.DOTALL)
                if style:
                    analysis["style"] = self._parse_list(style[0])
            except Exception:
                st.warning("Error parsing style section")
                
            try:
                performance = re.findall(r"PERFORMANCE:\n(.*?)(?=FIXED_CODE:)", review_text, re.DOTALL)
                if performance:
                    analysis["performance"] = self._parse_list(performance[0])
            except Exception:
                st.warning("Error parsing performance section")
                
            try:
                fixed_code = re.findall(r"FIXED_CODE:\n(.*?)$", review_text, re.DOTALL)
                return analysis, fixed_code[0].strip() if fixed_code else ""
            except Exception:
                st.warning("Error parsing fixed code")
                return analysis, ""
                
        except Exception as e:
            st.error(f"Analysis error: {str(e)}")
            st.error(f"Stack trace: {traceback.format_exc()}")
            return {"bugs": [], "style": [], "performance": []}, ""
    
    def _parse_list(self, text: str) -> list:
        """Parse bullet points into a list."""
        return [item.strip().strip('- ') for item in text.strip().split('\n') if item.strip()]

def verify_setup():
    
    if not OPENAI_API_KEY:
        st.error("⚠️ No OpenAI API key found. Please add your API key to the .env file.")
        st.stop()
    
    try:
        reviewer = CodeReviewer()
        if not reviewer.test_api_key():
            st.error("⚠️ Invalid OpenAI API key. Please check your API key in the .env file.")
            st.stop()
    except Exception as e:
        st.error(f"⚠️ Setup verification failed: {str(e)}")
        st.stop()

def main():
    st.set_page_config(
        page_title="AI Code Reviewer Pro",
        page_icon="🔍",
        layout="wide",
        initial_sidebar_state="expanded"
    )
    
    # Verify setup before proceeding
    verify_setup()
    
    with st.sidebar:
        st.header("⚙️ Settings")
        model = st.selectbox(
            "Select Model",
            ["gpt-3.5-turbo", "gpt-4"],
            help="GPT-4 is more expensive but may provide better results"
        )
        
    
    code_input = st.text_area(
        "Enter your Python code:",
        value=sample_code,
        height=300,
        help="Paste your Python code here for review"
    )
    
    col1, col2, col3 = st.columns([1, 1, 2])
    with col1:
        analyze_button = st.button("🔍 Review Code", type="primary", use_container_width=True)
    with col2:
        clear_button = st.button("🗑️ Clear Code", use_container_width=True)
    
    if clear_button:
        st.session_state.code_input = ""
        st.experimental_rerun()
    
    if analyze_button and code_input:
        try:
            with st.spinner("🔄 Analyzing your code..."):
                reviewer = CodeReviewer(model=model)
                analysis, fixed_code = reviewer.analyze_code(code_input)
                
                # Display results in tabs
                tabs = st.tabs(["🐛 Bugs", "🎨 Style", "⚡ Performance", "✅ Fixed Code"])
                
                with tabs[0]:
                    if analysis["bugs"]:
                        for bug in analysis["bugs"]:
                            st.error(bug)
                    else:
                        st.success("No bugs found! 🎉")
                
                with tabs[1]:
                    if analysis["style"]:
                        for suggestion in analysis["style"]:
                            st.info(suggestion)
                    else:
                        st.success("Code style looks good! 👍")
                
                with tabs[2]:
                    if analysis["performance"]:
                        for tip in analysis["performance"]:
                            st.warning(tip)
                    else:
                        st.success("No performance issues detected! 🚀")
                
                with tabs[3]:
                    if fixed_code:
                        st.code(fixed_code, language="python")
                        if st.button("📋 Copy Fixed Code"):
                            st.toast("Code copied to clipboard!")
                    else:
                        st.info("No improvements suggested for the code.")
                        
        except Exception as e:
            st.error(f"An error occurred: {str(e)}")
            st.error("Please check your API key and try again.")

if __name__ == "__main__":
    main()
