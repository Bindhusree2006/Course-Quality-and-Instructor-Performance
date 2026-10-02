# Course-Quality-and-Instructor-Performance
feedback of the courses and instructors
import streamlit as st
import pandas as pd
import numpy as np
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime, timedelta

# ==========================================
# 1. PAGE CONFIGURATION
# ==========================================
st.set_page_config(
    page_title="Academic Quality & Faculty Analytics",
    page_icon="🎓",
    layout="wide",
    initial_sidebar_state="expanded",
)

# ==========================================
# 2. DATA GENERATOR (ROBUST)
# ==========================================
@st.cache_data
def get_data(num_records=400):
    np.random.seed(42)
    departments = ["Computer Science", "Business & Finance", "Data Science", "Design", "Engineering"]
    instructors = {
        "Computer Science": ["Dr. Alan Turing", "Prof. Grace Hopper", "Dr. Linus V."],
        "Business & Finance": ["Prof. Warren B.", "Dr. Janet Y.", "Prof. Christine L."],
        "Data Science": ["Dr. Andrew N.", "Dr. Fei-Fei L.", "Prof. Yann L."],
        "Design": ["Prof. Don Norman", "Dr. Jony I."],
        "Engineering": ["Dr. Nikola T.", "Prof. Hedy L."]
    }
    courses = {
        "Computer Science": ["Algorithms & DS", "Cloud Architecture", "Operating Systems"],
        "Business & Finance": ["Corporate Valuation", "Financial Modeling", "Strategic Management"],
        "Data Science": ["Deep Learning", "Applied Statistics", "NLP Foundations"],
        "Design": ["UX Systems Design", "Interaction Architecture"],
        "Engineering": ["Robotics Automation", "Signal Processing"]
    }

    records = []
    base_date = datetime(2025, 1, 1)

    for i in range(num_records):
        dept = np.random.choice(departments)
        inst = np.random.choice(instructors[dept])
        course = np.random.choice(courses[dept])
        date = base_date + timedelta(days=int(np.random.randint(0, 360)))
        
        clarity = round(float(np.random.uniform(3.2, 5.0)), 2)
        responsiveness = round(float(np.random.uniform(2.8, 5.0)), 2)
        pedagogy = round(float(np.random.uniform(3.0, 5.0)), 2)
        course_depth = round(float(np.random.uniform(3.0, 5.0)), 2)
        practical_value = round(float(np.random.uniform(2.9, 5.0)), 2)
        difficulty = np.random.choice(["Introductory", "Intermediate", "Advanced"], p=[0.25, 0.5, 0.25])
        
        completion = round(float(np.random.uniform(65.0, 98.0)), 1)
        nps = int(np.random.choice(range(1, 11), p=[0.02, 0.03, 0.05, 0.05, 0.05, 0.1, 0.15, 0.2, 0.2, 0.15]))
        sentiment = "Positive" if nps >= 8 else ("Neutral" if nps in [6, 7] else "Negative")

        records.append({
            "Feedback_ID": f"FB-{1000 + i}",
            "Date": date,
            "Month": date.strftime("%Y-%m"),
            "Department": dept,
            "Instructor": inst,
            "Course": course,
            "Clarity": clarity,
            "Responsiveness": responsiveness,
            "Pedagogy": pedagogy,
            "Course_Depth": course_depth,
            "Practical_Value": practical_value,
            "Instructor_Score": round((clarity + responsiveness + pedagogy) / 3.0, 2),
            "Course_Score": round((course_depth + practical_value) / 2.0, 2),
            "Completion_Rate": completion,
            "Difficulty": difficulty,
            "NPS_Score": nps,
            "Sentiment": sentiment
        })
    return pd.DataFrame(records)

df_raw = get_data()

# ==========================================
# 3. SIDEBAR FILTERS
# ==========================================
st.sidebar.title("🎛️ Analytics Controls")

# Date range selection
min_d = df_raw["Date"].min().date()
max_d = df_raw["Date"].max().date()
date_input = st.sidebar.date_input("Date Range", [min_d, max_d], min_value=min_d, max_value=max_d)

# Department filter
all_depts = ["All"] + sorted(df_raw["Department"].unique().tolist())
selected_dept = st.sidebar.selectbox("Department", all_depts)

filtered_by_dept = df_raw if selected_dept == "All" else df_raw[df_raw["Department"] == selected_dept]

# Instructor & Course filter dependent on department
inst_list = ["All"] + sorted(filtered_by_dept["Instructor"].unique().tolist())
selected_instructor = st.sidebar.selectbox("Instructor", inst_list)

courses_list = ["All"] + sorted(filtered_by_dept["Course"].unique().tolist())
selected_course = st.sidebar.selectbox("Course", courses_list)

# Apply filters
df = df_raw.copy()

if isinstance(date_input, (list, tuple)) and len(date_input) == 2:
    start_date, end_date = date_input
    df = df[(df["Date"].dt.date >= start_date) & (df["Date"].dt.date <= end_date)]

if selected_dept != "All":
    df = df[df["Department"] == selected_dept]
if selected_instructor != "All":
    df = df[df["Instructor"] == selected_instructor]
if selected_course != "All":
    df = df[df["Course"] == selected_course]

# ==========================================
# 4. DASHBOARD HEADER & KPIS
# ==========================================
st.title("🎓 Course Quality & Instructor Performance")
st.caption("Live monitoring of academic delivery, student engagement, and sentiment.")

if df.empty:
    st.warning("⚠️ No data available matching the selected filters. Please expand your criteria.")
    st.stop()

# Compute metrics
promoters = (df["NPS_Score"] >= 9).sum()
detractors = (df["NPS_Score"] <= 6).sum()
nps_val = round(((promoters - detractors) / len(df)) * 100, 1)

kpi1, kpi2, kpi3, kpi4, kpi5 = st.columns(5)
kpi1.metric("Avg Instructor Score", f"{df['Instructor_Score'].mean():.2f} / 5.0")
kpi2.metric("Avg Course Score", f"{df['Course_Score'].mean():.2f} / 5.0")
kpi3.metric("Net Promoter Score", f"{nps_val}")
kpi4.metric("Avg Completion Rate", f"{df['Completion_Rate'].mean():.1f}%")
kpi5.metric("Total Reviews", f"{len(df):,}")

st.divider()

# ==========================================
# 5. TABS & VISUALIZATIONS
# ==========================================
tab_overview, tab_inst, tab_course, tab_data = st.tabs([
    "📈 Strategic Overview",
    "👨‍🏫 Instructor Analysis",
    "📚 Course Quality",
    "📑 Raw Data"
])

# ------------------------------------------
# TAB 1: OVERVIEW
# ------------------------------------------
with tab_overview:
    c1, c2 = st.columns(2)

    with c1:
        # Quadrant Chart
        quad = df.groupby(["Course", "Instructor"]).agg(
            Course_Score=("Course_Score", "mean"),
            Instructor_Score=("Instructor_Score", "mean"),
            Evaluations=("Feedback_ID", "count")
        ).reset_index()

        fig_quad = px.scatter(
            quad,
            x="Course_Score",
            y="Instructor_Score",
            size="Evaluations",
            color="Instructor_Score",
            color_continuous_scale="Viridis",
            hover_name="Course",
            hover_data=["Instructor", "Course_Score", "Instructor_Score"],
            title="Course Quality vs. Instructor Effectiveness",
            labels={"Course_Score": "Course Rating (1-5)", "Instructor_Score": "Instructor Rating (1-5)"}
        )
        fig_quad.add_hline(y=4.0, line_dash="dash", line_color="gray")
        fig_quad.add_vline(x=4.0, line_dash="dash", line_color="gray")
        fig_quad.update_layout(template="plotly_white")
        st.plotly_chart(fig_quad)

    with c2:
        # Monthly Performance Trend
        monthly = df.groupby("Month")[["Instructor_Score", "Course_Score"]].mean().reset_index()
        fig_trend = go.Figure()
        fig_trend.add_trace(go.Scatter(x=monthly["Month"], y=monthly["Instructor_Score"], mode="lines+markers", name="Instructor Avg"))
        fig_trend.add_trace(go.Scatter(x=monthly["Month"], y=monthly["Course_Score"], mode="lines+markers", name="Course Avg", line=dict(dash="dot")))
        fig_trend.update_layout(
            title="Monthly Quality Trajectory",
            yaxis=dict(range=[2.5, 5.0], title="Score"),
            xaxis=dict(title="Month"),
            template="plotly_white",
            legend=dict(orientation="h", y=1.1)
        )
        st.plotly_chart(fig_trend)

    # Department Benchmark
    dept_stats = df.groupby("Department")[["Instructor_Score", "Course_Score"]].mean().reset_index()
    fig_dept = px.bar(
        dept_stats,
        x="Department",
        y=["Instructor_Score", "Course_Score"],
        barmode="group",
        title="Departmental Rating Comparison",
        labels={"value": "Score (Out of 5)", "variable": "Metric"}
    )
    fig_dept.update_layout(template="plotly_white")
    st.plotly_chart(fig_dept)

# ------------------------------------------
# TAB 2: INSTRUCTOR EVALUATION
# ------------------------------------------
with tab_inst:
    col_rad, col_tbl = st.columns([1, 1])

    with col_rad:
        st.subheader("Teaching Competency Radar")
        metrics = ["Clarity", "Responsiveness", "Pedagogy", "Course_Depth", "Practical_Value"]
        values = [float(df[m].mean()) for m in metrics]
        values.append(values[0])  # Close loop
        cats = metrics + [metrics[0]]

        fig_radar = go.Figure(
            data=[
                go.Scatterpolar(
                    r=values,
                    theta=cats,
                    fill='toself',
                    name='Average Score'
                )
            ]
        )
        fig_radar.update_layout(
            polar=dict(radialaxis=dict(visible=True, range=[1, 5])),
            showlegend=False,
            template="plotly_white"
        )
        st.plotly_chart(fig_radar)

    with col_tbl:
        st.subheader("Faculty Leaderboard")
        inst_summary = df.groupby("Instructor").agg(
            Avg_Rating=("Instructor_Score", "mean"),
            Clarity=("Clarity", "mean"),
            Responsiveness=("Responsiveness", "mean"),
            Reviews=("Feedback_ID", "count")
        ).reset_index().sort_values(by="Avg_Rating", ascending=False)

        inst_summary["Avg_Rating"] = inst_summary["Avg_Rating"].round(2)
        inst_summary["Clarity"] = inst_summary["Clarity"].round(2)
        inst_summary["Responsiveness"] = inst_summary["Responsiveness"].round(2)

        st.dataframe(inst_summary, height=350)

# ------------------------------------------
# TAB 3: COURSE QUALITY
# ------------------------------------------
with tab_course:
    col_pie, col_box = st.columns(2)

    with col_pie:
        fig_sentiment = px.pie(
            df,
            names="Sentiment",
            title="Student Sentiment Breakdown",
            color="Sentiment",
            color_discrete_map={"Positive": "#22c55e", "Neutral": "#f59e0b", "Negative": "#ef4444"}
        )
        fig_sentiment.update_layout(template="plotly_white")
        st.plotly_chart(fig_sentiment)

    with col_box:
        fig_diff = px.box(
            df,
            x="Difficulty",
            y="Completion_Rate",
            color="Difficulty",
            category_orders={"Difficulty": ["Introductory", "Intermediate", "Advanced"]},
            title="Completion Rate by Difficulty Tier"
        )
        fig_diff.update_layout(template="plotly_white", showlegend=False)
        st.plotly_chart(fig_diff)

# ------------------------------------------
# TAB 4: RAW DATA & EXPORT
# ------------------------------------------
with tab_data:
    st.subheader("Feedback Records")
    
    search = st.text_input("Search (Instructor, Course, ID):")
    display_df = df.copy()
    if search:
        search_lower = search.lower()
        display_df = display_df[
            display_df["Course"].str.lower().str.contains(search_lower) |
            display_df["Instructor"].str.lower().str.contains(search_lower) |
            display_df["Feedback_ID"].str.lower().str.contains(search_lower)
        ]

    st.dataframe(display_df, height=380)

    csv_bytes = display_df.to_csv(index=False).encode("utf-8")
    st.download_button(
        label="📥 Download Filtered Data as CSV",
        data=csv_bytes,
        file_name="evaluation_data.csv",
        mime="text/csv"
    )
