import logging
import pandas as pd
import datetime as dt
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ConversationHandler, ContextTypes, PollAnswerHandler

logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

NOME, DURATA, DATA, PARTECIPANTI, ALTRO = range(5)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    keyboard = [
        [InlineKeyboardButton("Nuova Attività", callback_data='nuova_attivita')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text('Ciao! Scegli un\'opzione:', reply_markup=reply_markup)

async def nuova_attivita(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    query = update.callback_query
    await query.answer()
    await query.edit_message_text(text="Inserisci il nome dell'attività:")
    return NOME

async def nome(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['nome'] = update.message.text
    await update.message.reply_text('Inserisci la durata dell\'attività (in ore):')
    return DURATA

async def durata(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['durata'] = update.message.text
    await update.message.reply_text('Inserisci la data dell\'attività (gg-mm-aaaa):')
    return DATA

async def data(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    context.user_data['data'] = update.message.text
    options = ["Marco", "Juri", "Altro"]
    poll = await context.bot.send_poll(
        chat_id=update.effective_chat.id,
        question="Seleziona i partecipanti o premi Altro se ci sono altri:",
        options=options,
        is_anonymous=False,
        allows_multiple_answers=True
    )
    context.user_data['poll_id'] = poll.poll.id
    context.user_data['chat_id'] = update.effective_chat.id  # Store the chat ID
    context.bot_data['poll_options'] = options  # Store poll options for reference
    return PARTECIPANTI

async def poll_answer(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    poll_answer = update.poll_answer
    poll_id = context.user_data.get('poll_id')

    if poll_answer.poll_id != poll_id:
        return

    selected_options = [context.bot_data['poll_options'][i] for i in poll_answer.option_ids]
    context.user_data['partecipanti'] = selected_options
    chat_id = context.user_data.get('chat_id')  # Retrieve the stored chat ID

    if "Altro" in selected_options:
        await context.bot.send_message(
            chat_id=chat_id,
            text="Inserisci i nomi degli altri partecipanti, separati da uno spazio:"
        )
        context.user_data['awaiting_altro'] = True
        return ALTRO
    else:
        context.user_data['awaiting_altro'] = False
        await dati(update, context, chat_id)
        return ConversationHandler.END

async def altro(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if context.user_data.get('awaiting_altro'):
        partecipanti_in_piu = update.message.text.split()
        context.user_data['partecipanti'].extend(partecipanti_in_piu)
        context.user_data['awaiting_altro'] = False
        chat_id = context.user_data.get('chat_id')  # Retrieve the stored chat ID
        await dati(update, context, chat_id)
        return ConversationHandler.END
    return ConversationHandler.END

async def dati(update: Update, context: ContextTypes.DEFAULT_TYPE, chat_id: int) -> int:
    attivita = {
        'Nome': context.user_data['nome'],
        'Durata': context.user_data['durata'],
        'Data': context.user_data['data'],
        'Partecipanti': ", ".join(context.user_data['partecipanti'])
    }

    new_df = pd.DataFrame([attivita])

    if 'attivita_df' in context.user_data:
        context.user_data['attivita_df'] = pd.concat([context.user_data['attivita_df'], new_df], ignore_index=True)
    else:
        context.user_data['attivita_df'] = new_df

    keyboard = [
        [InlineKeyboardButton("SI", callback_data='conferma_attivita'), InlineKeyboardButton("NO", callback_data='annulla_attivita')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await context.bot.send_message(
        chat_id=chat_id,
        text=(
            f"Attività creata:\nNome: {context.user_data['nome']}\n"
            f"Durata: {context.user_data['durata']} ore\nData: {context.user_data['data']}\n"
            f"Partecipanti: {', '.join(context.user_data['partecipanti'])}\nConfermi?"
        ),
        reply_markup=reply_markup
    )
    return ConversationHandler.END

async def conferma_attivita(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    if 'attivita_df' in context.user_data:
        file_path = r"X:\Python\Progetti\P.E.R.E.N.N.I.O\attivita.csv"
        context.user_data['attivita_df'].to_csv(file_path, mode='a', header=not pd.io.common.file_exists(file_path), index=False)
        await context.bot.send_message(chat_id=update.effective_chat.id, text='Attività salvata')
    else:
        await context.bot.send_message(chat_id=update.effective_chat.id, text='Attività non esiste')
    return ConversationHandler.END

async def annulla_attivita(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await context.bot.send_message(chat_id=update.effective_chat.id, text='Operazione annullata.')
    return ConversationHandler.END

async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE) -> int:
    await update.message.reply_text('Operazione annullata.')
    return ConversationHandler.END

def main() -> None:
    token = '7118894338:AAGvOy5Tmbwqml52MlKbfHBPPvGwBDckmrM'
    application = Application.builder().token(token).build()

    conv_handler = ConversationHandler(
        entry_points=[CallbackQueryHandler(nuova_attivita, pattern='^nuova_attivita$')],
        states={
            NOME: [MessageHandler(filters.TEXT & ~filters.COMMAND, nome)],
            DURATA: [MessageHandler(filters.TEXT & ~filters.COMMAND, durata)],
            DATA: [MessageHandler(filters.TEXT & ~filters.COMMAND, data)],
            PARTECIPANTI: [PollAnswerHandler(poll_answer)],
            ALTRO: [MessageHandler(filters.TEXT & ~filters.COMMAND, altro)],
        },
        fallbacks=[CommandHandler('cancel', cancel)],
        per_user=True,
        per_chat=False
    )

    application.add_handler(CommandHandler("start", start))
    application.add_handler(conv_handler)
    application.add_handler(CallbackQueryHandler(conferma_attivita, pattern='^conferma_attivita$'))
    application.add_handler(CallbackQueryHandler(annulla_attivita, pattern='^annulla_attivita$'))

    application.run_polling()

if __name__ == '__main__':
    main()
