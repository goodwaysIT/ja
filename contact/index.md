---
layout: page
title: お問い合わせ
description: コンサルテーションやお問い合わせについて、私たちのチームにご連絡ください。
---

<div style="max-width: 1000px; margin: 0 auto; padding: 2rem 0;">
  <h1 style="color: #0055a4; margin-bottom: 1.5rem; text-align: center; font-size: 2.2rem;">Goodways ITチームへのお問い合わせ</h1>
  
  <p style="text-align: center; margin-bottom: 2.5rem; font-size: 1.1rem; color: #555;">お問い合わせをお待ちしております！私たちのサービスについてのご質問、コンサルテーションのご依頼、またはOracleデータベースのニーズについて私たちがどのようにお手伝いできるかについてご相談がありましたら、私たちのチームがサポートいたします。</p>

  <div style="display: flex; flex-wrap: wrap; gap: 2rem; justify-content: center; margin-bottom: 3rem;">
    <div style="flex: 1; min-width: 280px; background-color: #f9f9f9; border-radius: 8px; padding: 1.5rem; box-shadow: 0 2px 10px rgba(0,0,0,0.05);">
      <div style="display: flex; align-items: center; margin-bottom: 1rem;">
        <div style="width: 50px; height: 50px; background-color: #0055a4; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin-right: 1rem;">
          <i class="fas fa-envelope" style="color: white; font-size: 1.2rem;"></i>
        </div>
        <h3 style="color: #333; margin: 0;">メールでのお問い合わせ</h3>
      </div>
      <p style="margin-bottom: 0.5rem; color: #555;">一般的なお問い合わせ: <a href="mailto:info@goodways.co.jp" style="color: #0055a4; font-weight: 500;">info@goodways.co.jp</a></p>
      <p style="color: #555;">技術サポート: <a href="mailto:support@goodways.co.jp" style="color: #0055a4; font-weight: 500;">support@goodways.co.jp</a></p>
    </div>
    
    <div style="flex: 1; min-width: 280px; background-color: #f9f9f9; border-radius: 8px; padding: 1.5rem; box-shadow: 0 2px 10px rgba(0,0,0,0.05);">
      <div style="display: flex; align-items: center; margin-bottom: 1rem;">
        <div style="width: 50px; height: 50px; background-color: #0055a4; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin-right: 1rem;">
          <i class="fas fa-phone-alt" style="color: white; font-size: 1.2rem;"></i>
        </div>
        <h3 style="color: #333; margin: 0;">お電話でのお問い合わせ</h3>
      </div>
      <p style="margin-bottom: 0.5rem; color: #555;">電話番号: +81-XX-XXXX-XXXX</p>
      <p style="color: #555;">営業時間: 月曜日〜金曜日、9:00-18:00 JST</p>
    </div>
  </div>

<div style="margin-top: 2rem; background-color: #f9f9f9; border-radius: 10px; padding: 2rem; box-shadow: 0 3px 15px rgba(0,0,0,0.08);">
  <h2 style="color: #0055a4; text-align: center; margin-bottom: 1.5rem; font-size: 1.8rem; position: relative;">
    <span style="position: relative; display: inline-block; z-index: 1;">メッセージを送信する</span>
    <span style="position: absolute; bottom: -5px; left: 50%; transform: translateX(-50%); width: 80px; height: 3px; background-color: #ff6600; z-index: 0;"></span>
  </h2>

  <form class="contact-form" action="https://formspree.io/YOUR_FORMSPREE_ENDPOINT" method="POST" style="max-width: 650px; margin: 0 auto;">
    <div style="display: flex; flex-wrap: wrap; gap: 1.5rem; margin-bottom: 1.5rem;">
      <div style="flex: 1; min-width: 250px;">
        <label for="name" style="display: block; margin-bottom: 0.5rem; font-weight: 500; color: #333;">お名前</label>
        <input type="text" id="name" name="name" required style="width: 100%; padding: 0.75rem; border: 1px solid #ddd; border-radius: 4px; font-size: 1rem; transition: border-color 0.3s; outline: none;">
      </div>
      
      <div style="flex: 1; min-width: 250px;">
        <label for="email" style="display: block; margin-bottom: 0.5rem; font-weight: 500; color: #333;">メールアドレス</label>
        <input type="email" id="email" name="_replyto" required style="width: 100%; padding: 0.75rem; border: 1px solid #ddd; border-radius: 4px; font-size: 1rem; transition: border-color 0.3s; outline: none;">
      </div>
    </div>
    
    <div style="margin-bottom: 1.5rem;">
      <label for="subject" style="display: block; margin-bottom: 0.5rem; font-weight: 500; color: #333;">件名</label>
      <input type="text" id="subject" name="subject" style="width: 100%; padding: 0.75rem; border: 1px solid #ddd; border-radius: 4px; font-size: 1rem; transition: border-color 0.3s; outline: none;">
    </div>
    
    <div style="margin-bottom: 1.5rem;">
      <label for="message" style="display: block; margin-bottom: 0.5rem; font-weight: 500; color: #333;">メッセージ</label>
      <textarea id="message" name="message" rows="6" required style="width: 100%; padding: 0.75rem; border: 1px solid #ddd; border-radius: 4px; font-size: 1rem; transition: border-color 0.3s; outline: none; resize: vertical;"></textarea>
    </div>
    
    <div style="display: flex; justify-content: center;">
      <button type="submit" style="background-color: #0055a4; color: white; border: none; padding: 0.75rem 2rem; border-radius: 4px; font-size: 1rem; font-weight: 500; cursor: pointer; transition: background-color 0.3s; display: inline-flex; align-items: center;">
        <i class="fas fa-paper-plane" style="margin-right: 0.5rem;"></i> 送信する
      </button>
    </div>
  </form>

</div>
