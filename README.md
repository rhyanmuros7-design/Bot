# Botpython
# bot.py  —  python-telegram-bot v20+  (rode no Render)
# Requisitos: pip install "python-telegram-bot[job-queue]==20.7" stripe
#
# Variáveis de ambiente (defina no Render):
#   BOT_TOKEN          -> token do BotFather
#   STRIPE_SECRET_KEY  -> sk_... do Stripe
#   STRIPE_SUCCESS_URL -> https://seu-site.onrender.com/sucesso
#   STRIPE_CANCEL_URL  -> https://seu-site.onrender.com/cancelar
#   PROVIDER_TOKEN     -> token do provedor de pagamentos do Telegram (Stripe Connect no BotFather)
#   COD_ATIVO          -> "true" para liberar pagamento na entrega
#   PIX_KEY            -> sua chave Pix (ex.: 21 966181776)

import os
import time
import stripe
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, LabeledPrice
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, MessageHandler,
    PreCheckoutQueryHandler, ContextTypes, filters,
)

BOT_TOKEN = os.environ["BOT_TOKEN"]
STRIPE_SECRET_KEY = os.environ.get("STRIPE_SECRET_KEY", "")
STRIPE_SUCCESS_URL = os.environ.get("STRIPE_SUCCESS_URL", "https://example.com/sucesso")
STRIPE_CANCEL_URL = os.environ.get("STRIPE_CANCEL_URL", "https://example.com/cancelar")
PROVIDER_TOKEN = os.environ.get("PROVIDER_TOKEN", "")
COD_ATIVO = os.environ.get("COD_ATIVO", "true").lower() == "true"
PIX_KEY = os.environ.get("PIX_KEY", "21 966181776")
PIX_EXPIRA = 600  # 10 minutos em segundos

stripe.api_key = STRIPE_SECRET_KEY

# Catálogo de pacotes (preços em BRL)
TIERS = {
    "ouro":     {"nome": "Ouro",     "preco": 60,  "desc": "Acesso essencial: canal, sinais diários, suporte."},
    "negocios": {"nome": "Negócios", "preco": 80,  "desc": "Tudo de Ouro + mentorias semanais + planilha."},
    "preto":    {"nome": "Preto",    "preco": 130, "desc": "Black VIP: sala exclusiva + análises 1:1."},
    "amex":     {"nome": "Amex",     "preco": 120, "desc": "Cartão Amex virtual + cashback 2%."},
    "infinito": {"nome": "Infinito", "preco": 100, "desc": "Acesso vitalício a todos os canais."},
    "platina":  {"nome": "Platina",  "preco": 80,  "desc": "Card Platina + benefícios + suporte prioritário."},
}

# Pix em memória: {chat_id: {"expira": timestamp, "tier": id}}
pix_abertos = {}


def menu_tiers():
    kb = [
        [InlineKeyboardButton("Ouro · R$60", callback_data="tier:ouro"),
         InlineKeyboardButton("Negócios · R$80", callback_data="tier:negocios")],
        [InlineKeyboardButton("Preto · R$130", callback_data="tier:preto"),
         InlineKeyboardButton("Amex · R$120", callback_data="tier:amex")],
        [InlineKeyboardButton("Infinito · R$100", callback_data="tier:infinito"),
         InlineKeyboardButton("Platina · R$80", callback_data="tier:platina")],
    ]
    return InlineKeyboardMarkup(kb)


def menu_comprar(tier_id):
    kb = [
        [InlineKeyboardButton("Pix (10 min)", callback_data=f"pix:{tier_id}"),
         InlineKeyboardButton("Stripe", callback_data=f"stripe:{tier_id}")],
        [InlineKeyboardButton("Telegram Payments", callback_data=f"tgpay:{tier_id}")],
    ]
    if COD_ATIVO:
        kb.append([InlineKeyboardButton("Pagamento na entrega (COD)", callback_data=f"cod:{tier_id}")])
    return InlineKeyboardMarkup(kb)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Bem-vindo à *Vault*!\nEscolha um pacote para começar:",
        reply_markup=menu_tiers(),
        parse_mode="Markdown",
    )


async def on_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    data = q.data
    chat_id = q.message.chat_id

    if data.startswith("tier:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        texto = (f"*{t['nome']} — R$ {t['preco']}*\n\n{t['desc']}\n\n"
                 f"Escolha a forma de pagamento:")
        await q.message.reply_text(texto, reply_markup=menu_comprar(tier_id), parse_mode="Markdown")

    elif data.startswith("pix:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        pix_abertos[chat_id] = {"expira": time.time() + PIX_EXPIRA, "tier": tier_id}
        # Gere o QR/Pix copia-e-cola no seu provedor (placeholder abaixo)
        payload = f"00020126360014BR.GOV.BCB.PIX0114+5511999999999..."
        await q.message.reply_text(
            f"✅ *Pix gerado — {t['nome']} R$ {t['preco']}*\n"
            f"⏱ Expira em 10 minutos.\n\n"
            f"🔑 Chave Pix: `{PIX_KEY}`\n"
            f"💰 Valor: R$ {t['preco']}\n\n"
            f"`{payload}`\n\n"
            f"Após o pagamento, o pedido é confirmado automaticamente.",
            parse_mode="Markdown",
        )
        context.job_queue.run_once(checa_expiracao_pix, PIX_EXPIRA, data=chat_id)

    elif data.startswith("stripe:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        url = criar_checkout_stripe(t)
        await q.message.reply_text(
            f"💳 Pague *{t['nome']} R$ {t['preco']}* no Stripe:\n{url}",
            parse_mode="Markdown",
        )

    elif data.startswith("tgpay:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        if not PROVIDER_TOKEN:
            await q.message.reply_text("⚠️ PROVIDER_TOKEN não configurado.")
            return
        await context.bot.send_invoice(
            chat_id=chat_id,
            title=f"{t['nome']} — R$ {t['preco']}",
            description=t["desc"],
            payload=f"tier:{tier_id}",
            provider_token=PROVIDER_TOKEN,
            start_parameter=tier_id,
            currency="BRL",
            prices=[LabeledPrice(label=t["nome"], amount=int(t["preco"] * 100))],
        )

    elif data.startswith("cod:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        await q.message.reply_text(
            f"🚚 Pedido *{t['nome']} R$ {t['preco']}* — *Aguardando pagamento na entrega*.\n"
            f"Confirme seu endereço e telefone. O entregador emite o recibo.",
            parse_mode="Markdown",
        )


def criar_checkout_stripe(tier):
    session = stripe.checkout.Session.create(
        payment_method_types=["card"],
        line_items=[{
            "price_data": {
                "currency": "brl",
                "product_data": {"name": tier["nome"]},
                "unit_amount": int(tier["preco"] * 100),
            },
            "quantity": 1,
        }],
        mode="payment",
        success_url=STRIPE_SUCCESS_URL,
        cancel_url=STRIPE_CANCEL_URL,
    )
    return session.url


async def checa_expiracao_pix(context):
    chat_id = context.job.data
    info = pix_abertos.get(chat_id)
    if info and time.time() >= info["expira"]:
        del pix_abertos[chat_id]
        await context.bot.send_message(chat_id, "⌛ Seu Pix expirou. Gere um novo no menu.")


# Telegram Payments: confirma pré-checkout
async def precheckout(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.pre_checkout_query.answer(ok=True)


# Telegram Payments: pagamento concluído
async def pagamento_ok(update: Update, context: ContextTypes.DEFAULT_TYPE):
    tier_id = update.message.successful_payment.invoice_payload.split(":")[1]
    t = TIERS[tier_id]
    await update.message.reply_text(
        f"✅ *Pagamento confirmado!*\nPedido: *{t['nome']} — R$ {t['preco']}*\n"
        f"Recibo: {update.message.successful_payment.provider_payment_charge_id}",
        parse_mode="Markdown",
    )


def main():
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(on_callback))
    app.add_handler(PreCheckoutQueryHandler(precheckout))
    app.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, pagamento_ok))

    # No Render, use webhook (recomendado em produção):
    # app.run_webhook(
    #     listen="0.0.0.0",
    #     port=int(os.environ.get("PORT", 8000)),
    #     url_path=BOT_TOKEN,
    #     webhook_url=f"https://seu-app.onrender.com/{BOT_TOKEN}",
    # )
    app.run_polling()


if __name__ == "__main__":
    main()python
# bot.py  —  python-telegram-bot v20+  (rode no Render)
# Requisitos: pip install "python-telegram-bot[job-queue]==20.7" stripe
#
# Variáveis de ambiente (defina no Render):
#   BOT_TOKEN          -> token do BotFather
#   STRIPE_SECRET_KEY  -> sk_... do Stripe
#   STRIPE_SUCCESS_URL -> https://seu-site.onrender.com/sucesso
#   STRIPE_CANCEL_URL  -> https://seu-site.onrender.com/cancelar
#   PROVIDER_TOKEN     -> token do provedor de pagamentos do Telegram (Stripe Connect no BotFather)
#   COD_ATIVO          -> "true" para liberar pagamento na entrega
#   PIX_KEY            -> sua chave Pix (ex.: 21 966181776)

import os
import time
import stripe
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, LabeledPrice
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler, MessageHandler,
    PreCheckoutQueryHandler, ContextTypes, filters,
)

BOT_TOKEN = os.environ["BOT_TOKEN"]
STRIPE_SECRET_KEY = os.environ.get("STRIPE_SECRET_KEY", "")
STRIPE_SUCCESS_URL = os.environ.get("STRIPE_SUCCESS_URL", "https://example.com/sucesso")
STRIPE_CANCEL_URL = os.environ.get("STRIPE_CANCEL_URL", "https://example.com/cancelar")
PROVIDER_TOKEN = os.environ.get("PROVIDER_TOKEN", "")
COD_ATIVO = os.environ.get("COD_ATIVO", "true").lower() == "true"
PIX_KEY = os.environ.get("PIX_KEY", "21 966181776")
PIX_EXPIRA = 600  # 10 minutos em segundos

stripe.api_key = STRIPE_SECRET_KEY

# Catálogo de pacotes (preços em BRL)
TIERS = {
    "ouro":     {"nome": "Ouro",     "preco": 60,  "desc": "Acesso essencial: canal, sinais diários, suporte."},
    "negocios": {"nome": "Negócios", "preco": 80,  "desc": "Tudo de Ouro + mentorias semanais + planilha."},
    "preto":    {"nome": "Preto",    "preco": 130, "desc": "Black VIP: sala exclusiva + análises 1:1."},
    "amex":     {"nome": "Amex",     "preco": 120, "desc": "Cartão Amex virtual + cashback 2%."},
    "infinito": {"nome": "Infinito", "preco": 100, "desc": "Acesso vitalício a todos os canais."},
    "platina":  {"nome": "Platina",  "preco": 80,  "desc": "Card Platina + benefícios + suporte prioritário."},
}

# Pix em memória: {chat_id: {"expira": timestamp, "tier": id}}
pix_abertos = {}


def menu_tiers():
    kb = [
        [InlineKeyboardButton("Ouro · R$60", callback_data="tier:ouro"),
         InlineKeyboardButton("Negócios · R$80", callback_data="tier:negocios")],
        [InlineKeyboardButton("Preto · R$130", callback_data="tier:preto"),
         InlineKeyboardButton("Amex · R$120", callback_data="tier:amex")],
        [InlineKeyboardButton("Infinito · R$100", callback_data="tier:infinito"),
         InlineKeyboardButton("Platina · R$80", callback_data="tier:platina")],
    ]
    return InlineKeyboardMarkup(kb)


def menu_comprar(tier_id):
    kb = [
        [InlineKeyboardButton("Pix (10 min)", callback_data=f"pix:{tier_id}"),
         InlineKeyboardButton("Stripe", callback_data=f"stripe:{tier_id}")],
        [InlineKeyboardButton("Telegram Payments", callback_data=f"tgpay:{tier_id}")],
    ]
    if COD_ATIVO:
        kb.append([InlineKeyboardButton("Pagamento na entrega (COD)", callback_data=f"cod:{tier_id}")])
    return InlineKeyboardMarkup(kb)


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "👋 Bem-vindo à *Vault*!\nEscolha um pacote para começar:",
        reply_markup=menu_tiers(),
        parse_mode="Markdown",
    )


async def on_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    data = q.data
    chat_id = q.message.chat_id

    if data.startswith("tier:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        texto = (f"*{t['nome']} — R$ {t['preco']}*\n\n{t['desc']}\n\n"
                 f"Escolha a forma de pagamento:")
        await q.message.reply_text(texto, reply_markup=menu_comprar(tier_id), parse_mode="Markdown")

    elif data.startswith("pix:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        pix_abertos[chat_id] = {"expira": time.time() + PIX_EXPIRA, "tier": tier_id}
        # Gere o QR/Pix copia-e-cola no seu provedor (placeholder abaixo)
        payload = f"00020126360014BR.GOV.BCB.PIX0114+5511999999999..."
        await q.message.reply_text(
            f"✅ *Pix gerado — {t['nome']} R$ {t['preco']}*\n"
            f"⏱ Expira em 10 minutos.\n\n"
            f"🔑 Chave Pix: `{PIX_KEY}`\n"
            f"💰 Valor: R$ {t['preco']}\n\n"
            f"`{payload}`\n\n"
            f"Após o pagamento, o pedido é confirmado automaticamente.",
            parse_mode="Markdown",
        )
        context.job_queue.run_once(checa_expiracao_pix, PIX_EXPIRA, data=chat_id)

    elif data.startswith("stripe:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        url = criar_checkout_stripe(t)
        await q.message.reply_text(
            f"💳 Pague *{t['nome']} R$ {t['preco']}* no Stripe:\n{url}",
            parse_mode="Markdown",
        )

    elif data.startswith("tgpay:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        if not PROVIDER_TOKEN:
            await q.message.reply_text("⚠️ PROVIDER_TOKEN não configurado.")
            return
        await context.bot.send_invoice(
            chat_id=chat_id,
            title=f"{t['nome']} — R$ {t['preco']}",
            description=t["desc"],
            payload=f"tier:{tier_id}",
            provider_token=PROVIDER_TOKEN,
            start_parameter=tier_id,
            currency="BRL",
            prices=[LabeledPrice(label=t["nome"], amount=int(t["preco"] * 100))],
        )

    elif data.startswith("cod:"):
        tier_id = data.split(":")[1]
        t = TIERS[tier_id]
        await q.message.reply_text(
            f"🚚 Pedido *{t['nome']} R$ {t['preco']}* — *Aguardando pagamento na entrega*.\n"
            f"Confirme seu endereço e telefone. O entregador emite o recibo.",
            parse_mode="Markdown",
        )


def criar_checkout_stripe(tier):
    session = stripe.checkout.Session.create(
        payment_method_types=["card"],
        line_items=[{
            "price_data": {
                "currency": "brl",
                "product_data": {"name": tier["nome"]},
                "unit_amount": int(tier["preco"] * 100),
            },
            "quantity": 1,
        }],
        mode="payment",
        success_url=STRIPE_SUCCESS_URL,
        cancel_url=STRIPE_CANCEL_URL,
    )
    return session.url


async def checa_expiracao_pix(context):
    chat_id = context.job.data
    info = pix_abertos.get(chat_id)
    if info and time.time() >= info["expira"]:
        del pix_abertos[chat_id]
        await context.bot.send_message(chat_id, "⌛ Seu Pix expirou. Gere um novo no menu.")


# Telegram Payments: confirma pré-checkout
async def precheckout(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.pre_checkout_query.answer(ok=True)


# Telegram Payments: pagamento concluído
async def pagamento_ok(update: Update, context: ContextTypes.DEFAULT_TYPE):
    tier_id = update.message.successful_payment.invoice_payload.split(":")[1]
    t = TIERS[tier_id]
    await update.message.reply_text(
        f"✅ *Pagamento confirmado!*\nPedido: *{t['nome']} — R$ {t['preco']}*\n"
        f"Recibo: {update.message.successful_payment.provider_payment_charge_id}",
        parse_mode="Markdown",
    )


def main():
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CallbackQueryHandler(on_callback))
    app.add_handler(PreCheckoutQueryHandler(precheckout))
    app.add_handler(MessageHandler(filters.SUCCESSFUL_PAYMENT, pagamento_ok))

    # No Render, use webhook (recomendado em produção):
    # app.run_webhook(
    #     listen="0.0.0.0",
    #     port=int(os.environ.get("PORT", 8000)),
    #     url_path=BOT_TOKEN,
    #     webhook_url=f"https://seu-app.onrender.com/{BOT_TOKEN}",
    # )
    app.run_polling()


if __name__ == "__main__":
    main()
