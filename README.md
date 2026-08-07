import asyncio
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import (
    ApplicationBuilder,
    ContextTypes,
    MessageHandler,
    CallbackQueryHandler,
    filters
)

# ==================== CONFIGURACIÓN ====================
TOKEN = "8993585594:AAG80gO-6uNxJKiwtxGYjAr0CWdnerYTHSM"

# El enlace de invitación de tu grupo (para que se comparta)
ENLACE_INVITACION = "https://t.me/+pgNfbQos9QZkYzFh"

# El enlace del OTRO grupo/canal a donde quieres que los mande al hacer clic en "Ya compartí"
ENLACE_OTRO_GRUPO = "https://t.me/+DDAAJYnv6F1mM2Fh"

# Diccionario para recordar el último mensaje de bienvenida por chat
ultimos_mensajes_bienvenida = {}
# ========================================================

# 1. NUEVO MIEMBRO: Silenciar y mostrar los dos botones
async def nuevo_miembro(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        await update.message.delete()
    except Exception:
        pass

    chat_id = update.effective_chat.id

    for member in update.message.new_chat_members:
        if member.id == context.bot.id:
            continue
        
        user_id = member.id
        nombre_usuario = member.first_name

        # Restringir permisos para que no pueda hablar hasta desbloquear
        try:
            await context.bot.restrict_chat_member(
                chat_id=chat_id,
                user_id=user_id,
                permissions={
                    "can_send_messages": False,
                    "can_send_media_messages": False,
                    "can_send_polls": False,
                    "can_send_other_messages": False
                }
            )
        except Exception:
            pass

        # Borrar mensaje de bienvenida anterior si existe en el chat
        if chat_id in ultimos_mensajes_bienvenida:
            try:
                msg_anterior_id = ultimos_mensajes_bienvenida[chat_id]
                await context.bot.delete_message(chat_id=chat_id, message_id=msg_anterior_id)
            except Exception:
                pass

        # Estructura de los dos botones:
        # 1. Botón "Compartir" arriba (abre el enlace de Telegram para compartir)
        # 2. Botón "Ya comparti" abajo (es un enlace web que lleva al otro grupo y ejecuta la verificación)
        teclado = [
            [InlineKeyboardButton("📤 Compartir", url=f"https://t.me/share/url?url={ENLACE_INVITACION}&text=¡Únete%a%este%20excelente%grupoPACKSEC2.0!")],
            [InlineKeyboardButton("✅ Ya compartí", callback_data=f"desbloquear_{user_id}")]
        ]
        reply_markup = InlineKeyboardMarkup(teclado)

        mensaje = (
            f"👋 ¡Hola Te Saluda TIO ORO bienvenido/a!\n\n"
            f"🔒 **CONTENIDO BLOQUEADO PACKS CON PERFILES SOLO ECUADOR**\n"
            f"Para poder hablar y ver el contenido de este grupo, debes compartir el enlace en otros grupos:\n\n"
            f"1️⃣ Haz clic en **📤 Compartir** y envíalo a 2 grupos.\n"
            f"2️⃣ Luego haz clic en **✅ Ya compartí** para verificar tu acceso y unirte al respaldo.\n\n"
            f"*(Tienes 3 minutos para completar el proceso o serás expulsado)*"
        )

        mensaje_bienvenida = await context.bot.send_message(
            chat_id=chat_id,
            text=mensaje,
            reply_markup=reply_markup,
            parse_mode="Markdown"
        )

        ultimos_mensajes_bienvenida[chat_id] = mensaje_bienvenida.message_id

        # Tarea para expulsar si pasan 3 minutos sin verificar
        async def expulsar_si_no_verifica():
            await asyncio.sleep(180)
            try:
                chat_member = await context.bot.get_chat_member(chat_id, user_id)
                if not chat_member.can_send_messages:
                    await context.bot.ban_chat_member(chat_id, user_id)
            except Exception:
                pass

        asyncio.create_task(expulsar_si_no_verifica())

# 2. MANEJAR CLIC EN EL BOTÓN "YA COMPARTÍ"
async def boton_desbloquear(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    data = query.data
    user_id = query.from_user.id

    if data.startswith("desbloquear_"):
        id_objetivo = int(data.split("_")[1])
        if user_id != id_objetivo:
            await query.answer("❌ Este botón no es para ti.", show_alert=True)
            return

        chat_id = update.effective_chat.id

        try:
            # Devolver permisos de escritura
            await context.bot.restrict_chat_member(
                chat_id=chat_id,
                user_id=user_id,
                permissions={
                    "can_send_messages": True,
                    "can_send_media_messages": True,
                    "can_send_polls": True,
                    "can_send_other_messages": True
                }
            )
            
            # Mensaje de éxito dándole también el link del otro grupo
            texto_exito = (
                f"🎉 ¡Verificación exitosa, {query.from_user.first_name}!\n\n"
                f"Ya puedes participar en este grupo.\n"
                f"🔗 Únete también a nuestro otro grupo aquí: {ENLACE_OTRO_GRUPO}"
            )
            
            await query.message.edit_text(texto_exito)
            
        except Exception:
            pass

if __name__ == '__main__':
    app = ApplicationBuilder().token(TOKEN).build()

    app.add_handler(MessageHandler(filters.StatusUpdate.NEW_CHAT_MEMBERS, nuevo_miembro))
    app.add_handler(CallbackQueryHandler(boton_desbloquear, pattern="^desbloquear_"))

    print("🚀 Bot con botones de compartir y verificación ejecutándose...")
    app.run_polling()
