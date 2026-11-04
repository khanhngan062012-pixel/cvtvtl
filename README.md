# 1. Cài đặt các thư viện cần thiết
!pip install gradio SpeechRecognition google-genai -qq

import gradio as gr
import speech_recognition as sr
from google import genai
import datetime
from zoneinfo import ZoneInfo

# 🔑 API Key của bạn
API_KEY = "AQ.Ab8RN6JeMAyuE6FQGQh9gvTDLEkcOfhQMcNRnmkNKhis0oWDyQ"
client = genai.Client(api_key=API_KEY)

history_db = []

def start_loading():
    return gr.update(value="⏳ Sanrio đang lắng nghe... (Chờ xíu nha)", interactive=False)

def process_psychology(audio_filepath):
    global history_db

    if audio_filepath is None:
        return (
            gr.update(visible=True),
            gr.update(visible=False),
            gr.update(visible=False),
            "⚠️ Bạn ơi, hãy bấm nút micro để tâm sự với tớ nhé!",
            render_history(),
            gr.update(choices=get_choices(), value=get_default_choice()),
            gr.update(value="Gửi tâm sự cho Sanrio", interactive=True)
        )

    r = sr.Recognizer()
    try:
        with sr.AudioFile(audio_filepath) as source:
            audio_data = r.record(source)
            user_story = r.recognize_google(audio_data, language="vi-VN")
    except Exception as e:
        return (
            gr.update(visible=True),
            gr.update(visible=False),
            gr.update(visible=False),
            f"❌ Tớ chưa nghe rõ lắm, bạn nói lại to và rõ hơn xíu nha! (Lỗi: {e})",
            render_history(),
            gr.update(choices=get_choices(), value=get_default_choice()),
            gr.update(value="Gửi tâm sự cho Sanrio", interactive=True)
        )

    try:
        prompt = (
            f"Bạn là Sanrio - Chuyên viên tư vấn tâm lý ấm áp, thấu cảm và nhạy bén. "
            f"Cách xưng hô: Cậu -tớ."
            f"Thân chủ tâm sự: '{user_story}'. "
            f"Hãy phản hồi chi tiết theo cấu trúc 3 phần sau đây:\n"
            f"1. Tóm tắt sơ yếu lí lịch và biểu hiện bất ổn tâm lí của thân chủ\n"
            f"2. Phân tích cơ chế tâm lý.\n"
            f"3. Đưa lời khuyên\n"
        )

        response = client.models.generate_content(
            model='gemini-3.8-flash',
            contents=prompt
        )
        analysis_result = response.text

    except Exception as e:
        analysis_result = f"❌ Ôi, kết nối AI đang gặp trục trặc: {e}"

    # 🕒 Chuẩn giờ Việt Nam (UTC +7)
    vn_time = datetime.datetime.now(ZoneInfo("Asia/Ho_Chi_Minh"))
    now = vn_time.strftime("%H:%M - %d/%m/%Y")
    client_id = f"Thân chủ #{len(history_db) + 1} ({now})"

    history_db.append({
        "id": client_id,
        "story": user_story,
        "analysis": analysis_result
    })

    current_output = f"🗣️ Lời tâm sự của cậu: '{user_story}'\n\n✨ GÓC PHÂN TÍCH TỪ SANRIO ✨\n\n{analysis_result}"

    return (
        gr.update(visible=False),
        gr.update(visible=True),
        gr.update(visible=False),
        current_output,
        render_history(),
        gr.update(choices=get_choices(), value=get_default_choice()),
        gr.update(value="Gửi tâm sự cho Sanrio", interactive=True)
    )

def render_history():
    if not history_db:
        return "<div style='text-align: center; color: #444; font-size: 16px; padding: 20px; font-family: \"Montserrat\", sans-serif;'>Chưa có lịch sử tư vấn nào hết nè! Hãy tâm sự với Sanrio nha.</div>"

    html = "<div style='display: flex; flex-direction: column; gap: 12px;'>"
    for item in reversed(history_db):
        html += f"""
        <details class="custom-details" style="background-color: #ffffff; padding: 16px 18px; border-radius: 12px; border: 2px solid #ffd1dc; box-shadow: 0 2px 5px rgba(0,0,0,0.05);">
            <summary style="font-weight: 700; color: #d6336c !important; font-size: 16px; font-family: 'Montserrat', sans-serif; cursor: pointer; outline: none; display: flex; justify-content: space-between; align-items: center;">
                <span style="color: #d6336c !important;">👤 {item['id']}</span>
                <span style="font-size: 13px; color: #666; font-weight: 600;">(Bấm để xem chi tiết ▾)</span>
            </summary>
            <div style="margin-top: 12px; padding-top: 12px; border-top: 1px dashed #ffd1dc; font-size: 14px; color: #111111 !important; font-family: 'Montserrat', sans-serif; line-height: 1.6;">
                <p style="margin-bottom: 8px; color: #111111 !important;"><b>🗣️ Tâm sự của thân chủ:</b> "{item['story']}"</p>
                <div style="white-space: pre-line; color: #111111 !important; font-weight: 600; background-color: #fff5f7; padding: 12px; border-radius: 8px; border: 1px solid #ffccd5;"><b>📋 Phân tích chi tiết:</b>\n{item['analysis']}</div>
            </div>
        </details>
        """
    html += "</div>"
    return html

def get_choices():
    choices = [item['id'] for item in history_db]
    return choices if choices else []

def get_default_choice():
    choices = get_choices()
    return choices[0] if choices else None

def delete_single_item(selected_id):
    global history_db
    if selected_id:
        history_db = [item for item in history_db if item['id'] != selected_id]

    choices = get_choices()
    default_val = choices[0] if choices else None
    return render_history(), gr.update(choices=choices, value=default_val)

def clear_all_history():
    global history_db
    history_db = []
    return render_history(), gr.update(choices=[], value=None)

# ==================== GIAO DIỆN ====================
with gr.Blocks() as demo:

    gr.HTML("""
        <style>
            @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Montserrat:wght@300;400;500;600;700&display=swap');

            .gradio-container {
                background: #ffffff !important;
                font-family: 'Montserrat', 'Inter', sans-serif !important;
            }
            .block {
                background-color: #fff9fa !important;
                border: 2px solid #ffd1dc !important;
                border-radius: 16px !important;
            }
            .border-blue-500, label span, .text-gray-500, label {
                background: #ffd1dc !important;
                color: #222222 !important;
                font-weight: bold !important;
                border-radius: 8px !important;
                padding: 4px 10px !important;
                font-family: 'Montserrat', sans-serif !important;
                font-size: 15px !important;
            }
            /* 🔥 Ép chữ nhãn "Chọn thân chủ muốn xóa" sang màu đen đậm */
            label span, .block span, [data-testid="dropdown"] span {
                color: #111111 !important;
            }
            .custom-details *, .custom-details div, .custom-details p, .custom-details span {
                color: #111111 !important;
            }
            .custom-details summary span {
                color: #d6336c !important;
            }
            .dropdown input, .wrap select, [data-testid="dropdown"] input {
                color: #111111 !important;
                background-color: #ffffff !important;
                font-weight: 600 !important;
            }
            select option {
                color: #111111 !important;
                background-color: #ffffff !important;
            }
            button, .border, .bg-gray-200, [data-testid="state-button"] {
                background-color: #ffd1dc !important;
                color: #222222 !important;
                font-weight: bold !important;
                border-radius: 10px !important;
                font-family: 'Montserrat', sans-serif !important;
                font-size: 15px !important;
                border: none !important;
            }
            button.primary {
                background: linear-gradient(135deg, #fff0f2 0%, #ffd1dc 100%) !important;
                border: 1px solid #ffb6c1 !important;
                color: #d6336c !important;
                font-size: 16px !important;
                font-weight: bold !important;
                border-radius: 12px !important;
                box-shadow: 0 4px 10px rgba(255, 209, 220, 0.4) !important;
                font-family: 'Montserrat', sans-serif !important;
            }
            .dark-result-box textarea {
                background-color: #ffffff !important;
                color: #111111 !important;
                font-size: 16px !important;
                font-weight: 600 !important;
                border: 2px solid #ffd1dc !important;
                border-radius: 12px !important;
                font-family: 'Montserrat', sans-serif !important;
                line-height: 1.6 !important;
            }
        </style>
    """)

    # MÀN HÌNH 1
    with gr.Column(visible=True) as screen_1:
        gr.HTML("""
            <div style="text-align: center; background: #ffffff; padding: 35px; border-radius: 20px; color: #333; margin-bottom: 20px; border: 2px solid #ffd1dc;">
                <div style="font-size: 80px; font-weight: 700; color: #ff8fa3; line-height: 1; margin-bottom: 15px; font-family: 'Montserrat', sans-serif;">0v0</div>
                <h1 style="margin: 0; font-family: 'Montserrat', sans-serif; font-size: 30px; color: #222222; font-weight: 700;">Sanrio - Trợ Lý Tâm Lý của cậu</h1>
                <p style="margin: 8px 0 0 0; font-size: 16px; color: #555555; font-family: 'Montserrat', sans-serif; font-weight: 500;">Tớ ở đây để lắng nghe, thấu hiểu và ôm lấy những cảm xúc mà cậu muốn sẻ chia</p>
            </div>
        """)
        gr.HTML("<h3 style='text-align: center; color: #222222; font-family: \"Montserrat\", sans-serif; font-size: 20px; font-weight: 600;'>Bấm Micro để trò chuyện với Sanrio nhé:</h3>")
        audio_in = gr.Audio(sources=["microphone"], type="filepath", label="Bấm vào Micro để trò chuyện")

        with gr.Row():
            send_btn = gr.Button("Gửi tâm sự cho Sanrio", variant="primary", scale=2)
            to_history_btn1 = gr.Button("Xem Lịch Sử Thân Chủ", scale=1)

    # MÀN HÌNH 2
    with gr.Column(visible=False) as screen_2:
        gr.HTML("""
            <div style="text-align: center; margin-bottom: 20px;">
                <h3 style="color: #d6336c; font-family: 'Montserrat', sans-serif; font-size: 22px; font-weight: 700; margin: 0 0 8px 0;">✨ Kết quả phân tích dành riêng cho cậu đây ✨</h3>
                <p style="color: #666; font-size: 13px; font-style: italic; margin: 0; font-family: 'Montserrat', sans-serif;"> Lưu ý: Đây là định hướng tâm lý sơ bộ, không thay thế trị liệu y khoa chuyên sâu.</p>
            </div>
        """)
        result_box = gr.Textbox(label="", lines=13, interactive=False, elem_classes=["dark-result-box"])

        with gr.Row():
            back_btn = gr.Button("Quay lại", variant="primary", scale=2)
            to_history_btn2 = gr.Button("Xem Lịch Sử Tư Vấn", scale=1)

    # MÀN HÌNH 3
    with gr.Column(visible=False) as screen_3:
        gr.HTML("<h3 style='text-align: center; color: #d6336c; font-family: 'Montserrat', sans-serif; font-size: 22px; font-weight: 700; margin-bottom: 20px;'>📜 Hồ Sơ & Lịch Sử Tư Vấn Thân Chủ</h3>")
        history_display_box = gr.HTML(value="")

        with gr.Row():
            delete_dropdown = gr.Dropdown(label="Chọn Ô Lịch Sử Muốn Xóa", choices=[], interactive=True, scale=2)
            delete_one_btn = gr.Button("❌ Xóa ô này", variant="primary", scale=1)

        with gr.Row():
            back_from_history_btn = gr.Button("Quay lại", variant="primary", scale=2)
            clear_all_btn = gr.Button("Xóa Toàn Bộ Lịch Sử", scale=1)

    # Sự kiện
    send_btn.click(
        fn=start_loading,
        inputs=None,
        outputs=[send_btn]
    ).then(
        fn=process_psychology,
        inputs=[audio_in],
        outputs=[screen_1, screen_2, screen_3, result_box, history_display_box, delete_dropdown, send_btn]
    )

    to_history_btn1.click(fn=lambda: (gr.update(visible=False), gr.update(visible=False), gr.update(visible=True), render_history(), gr.update(choices=get_choices(), value=get_default_choice())), outputs=[screen_1, screen_2, screen_3, history_display_box, delete_dropdown])
    to_history_btn2.click(fn=lambda: (gr.update(visible=False), gr.update(visible=False), gr.update(visible=True), render_history(), gr.update(choices=get_choices(), value=get_default_choice())), outputs=[screen_1, screen_2, screen_3, history_display_box, delete_dropdown])

    back_from_history_btn.click(fn=lambda: (gr.update(visible=True), gr.update(visible=False), gr.update(visible=False)), outputs=[screen_1, screen_2, screen_3])
    back_btn.click(fn=lambda: (gr.update(visible=True), gr.update(visible=False), gr.update(visible=False)), outputs=[screen_1, screen_2, screen_3])

    delete_one_btn.click(
        fn=delete_single_item,
        inputs=[delete_dropdown],
        outputs=[history_display_box, delete_dropdown]
    )

    clear_all_btn.click(
        fn=clear_all_history,
        outputs=[history_display_box, delete_dropdown]
    )

demo.launch(inline=False, share=True)
