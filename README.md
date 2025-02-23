Sapius Bot - Problemas Atuais e Soluções

Descrição Geral

Sapius é um bot do Discord que utiliza a API da OpenAI para responder a interações e gerar imagens. O bot está funcional, mas apresenta um problema persistente relacionado ao contexto das respostas.

Problema Atual

Erro: O bot responde sempre ao mesmo contexto inicial em vez de utilizar todo o histórico da conversa para gerar respostas mais inteligentes.

Comportamento Observado:

O contexto antes da atualização está vazio ou contém poucas mensagens.

Após a atualização, o contexto mantém apenas um subconjunto fixo de mensagens.

O bot ignora interações anteriores e não consegue construir respostas contextuais adequadas.

Possível Causa:

A variável context pode estar a ser sobrescrita incorretamente em vez de ser acumulada.

O método de atualização do contexto pode estar a substituir o histórico em vez de adicionar novas mensagens.

Pode haver um problema na estrutura de armazenamento do contexto que impede o crescimento adequado da conversa.

Possível Solução

Verificar a Lógica de Atualização do Contexto:

Garantir que novas mensagens sejam adicionadas ao array do contexto e não apenas substituídas.

Testar a Persistência do Contexto:

Adicionar logs para verificar como o contexto está a ser armazenado após cada mensagem.

Ajustar a Lógica de Construção da Query para OpenAI:

Assegurar que todo o histórico relevante seja enviado na requisição para gerar respostas mais inteligentes.

Evitar Sobrescrita do Contexto:

Criar uma variável separada para armazenar o contexto atualizado antes de atribuí-lo à variável global.

//////////////////////////////--------------------------------------------------------- funcs ......... etc ------------------------- //////////////////////////////////////////////


// Função para gerar um hash único para o contexto
function generateContextHash(context) {
    const contextString = JSON.stringify(context); // Converter o contexto em string
    return crypto.createHash('sha256').update(contextString).digest('hex'); // Gerar o hash
}

// Função para obter uma resposta da OpenAI usando o contexto dinâmico (histórico de conversa)
async function getOpenAIResponse(channelId, message, newMessage) {
    try {
        console.log("--- NOVA INTERAÇÃO ---");
        console.log("Channel ID:", channelId);
        //Verifica se "Sapius" está na mensagem antes de processá-la
        if (!newMessage.toLowerCase().includes("sapius")) {
            console.log("Mensagem não contém 'Sapius', não processando.");
            return; // Ignorar mensagens que não mencionam "Sapius"
        }
        if (newMessage.toLowerCase().startsWith("gera uma imagem de") || newMessage.toLowerCase().startsWith("cria uma imagem de")) {
            const prompt = newMessage.replace(/(gera|cria) uma imagem de /i, "").trim();
            console.log("Gerando imagem com prompt:", prompt);
            await generateImage(prompt, message);
            return; // Evita que continue para o GPT De texto
        }
        let channelContext = channelContexts[channelId] || [];
        const currentContextHash = generateContextHash(channelContext); // Gerar o hash do contexto atual
        console.log("Context Before Update:", channelContext);

        const filteredMessage = filterNoise(newMessage);

        const newContext = [...channelContext];
        if (filteredMessage) {
            // Verificar se a mensagem não é redundante   
         const isDuplicate = newContext.some(msg => msg.content === filteredMessage);
            if (!isDuplicate) {
                newContext.push({ author: message.author.username, content: filteredMessage, timestamp: Date.now() });
            }
        }

        const updatedContextHash = generateContextHash(newContext); // Gerar o hash do contexto após a atualização
        console.log("Context After Update:", newContext);

        // 2. Verificar se o contexto foi realmente alterado
        if (currentContextHash === updatedContextHash) {
            console.log("Contexto não alterado, não será enviado para a API.");
            return; // Se o contexto não mudou, não envia para a API
        }

        console.log("Contexto alterado, enviando para a API.");
        console.log("Context After Update:", updatedContextHash);
        console.log("Context After Update:", channelContext);

        const weightedContext = weightRecentMessages(channelContext);
        const conversationPrompt = [
            { role: 'system', content: "Mantenha o contexto da conversa e responda no idioma utilizado pelo usuário." },
            { role: 'system', content: "--- INÍCIO DOS PROMPTS ---" },
            // Adicione aqui todos os seus prompts do sistema
            { role: 'system', content: "Adapte seu nível de detalhe à complexidade da pergunta." },
            { role: 'system', content: "Use exemplos práticos para ilustrar conceitos." },
            // ... (outros prompts)
            { role: 'system', content: "--- FIM DOS PROMPTS ---" },
            ...weightedContext.map(msg => ({ role: msg.author === message.author.username ? 'user' : 'assistant', content: msg.content })),
        ];

        console.log("Prompt enviado para a API:", conversationPrompt);

        const response = await axios.post(
            'https://api.openai.com/v1/chat/completions',
            {
                model: 'gpt-3.5-turbo',
                messages: conversationPrompt,
                max_tokens: 500,
            },
            {
                headers: {
                    'Authorization': `Bearer ${OPEN_API_KEY}`,
                    'Content-Type': 'application/json'
                }
            }
        );
        const responseText = response.data.choices[0].message.content;
        console.log("Texto da resposta gerada:", responseText);
        if (!responseText || responseText.trim() === '') {
            console.log("Empty response, not sending a message.");
            await sendLongMessage(message, 'Sorry, I couldn\'t generate a response. Try again later.');
        } else {
            await sendLongMessage(message, responseText);
        }
        // Atualizar o contexto guardado
        channelContexts[channelId] = channelContext;
        console.log(channelContexts);
    } catch (error) {
        console.error('Error calling the OpenAI API:', error);
        await sendLongMessage(message, 'I\'m overwhelmed! Try again later.');
    }
}
// Função para gerar um resumo do contexto
async function summarizeContext(messages) {
    try {
        const prompt = `Summarize the following conversation:\n\n${messages.map(msg => `${msg.author}: ${msg.content}`).join('\n')}\n\nSummary:`;
        const response = await axios.post(
            'https://api.openai.com/v1/completions',
            {
                model: 'text-davinci-003',
                prompt: prompt,
                max_tokens: 150,
            },
            {
                headers: {
                    'Authorization': `Bearer ${OPEN_API_KEY}`,
                    'Content-Type': 'application/json'
                }
            }
        );
        const summary = response.data.choices[0].text.trim();

        return [{ author: 'System', content: `Summary: ${summary}`, timestamp: Date.now() }];
    } catch (error) {
        console.error("Error summarizing context:", error);
        return messages.slice(-1);
    }
}

// Função para pesar as mensagens mais recentes
function weightRecentMessages(messages) {
    const now = Date.now();
    return messages.map((msg, index) => {
        const weight = Math.exp(-(now - msg.timestamp) / (CONTEXT_TIMEOUT / 3));
        return { ...msg, weight: weight * 2 };
    }).sort((a, b) => b.weight - a.weight);
}

// Função para filtrar "ruídos" no texto
function filterNoise(message) {
    if (message.trim().length === 0 || /^<@&\d+>/.test(message) || /^<@\d+>/.test(message) || /^https?:\/\/\S+$/.test(message) || /^:(\w+):$/.test(message)) {
        return null; // Ignora mensagens com menções, links, emojis ou vazias
    }
    return message;
}




// Evento `messageCreate`
client.on('messageCreate', async (message) => {
    try {
        if (message.system || message.author.id === client.user.id || processedMessages.has(message.id)) {
            return;
        }

        processedMessages.add(message.id);

        const channelId = message.channel.id;
        const userMessage = message.content;

        if (userMessage.trim().length > 0) {
            await message.channel.sendTyping();
            await getOpenAIResponse(channelId, message, userMessage);
        }
    } catch (error) {
        console.error("Error in messageCreate event:", error);
    }
});










/////////////////////////////////////////////////--------------------------Logs Relevantes-------------------------------///////////////////////////////////////////////




✅ Bot is online as Sapius#7448!
Successfully registered /ping command globally!
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Mensagem não contém 'Sapius', não processando.
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Context Before Update: (0) []
Context After Update: (1) [{…}]
Contexto alterado, enviando para a API.
Context After Update: c62ff6194c6591bbf16814d311be7f1075617af4683757d6f6fb5253c253c609
Context After Update: (0) []
Prompt enviado para a API: (5) [{…}, {…}, {…}, {…}, {…}]
Texto da resposta gerada: Olá! Em qual assunto você precisa de ajuda hoje?
{1331271723303309422: Array(0)}
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Context Before Update: (0) []
Context After Update: (1) [{…}]
Contexto alterado, enviando para a API.
Context After Update: 1a8e2167274a7543334f31e7813aa887d3c154426cbcc0d3754d31daec491e2c
Context After Update: (0) []
Prompt enviado para a API: (5) [{…}, {…}, {…}, {…}, {…}]
Texto da resposta gerada: Olá! Como posso ajudar você hoje?
{1331271723303309422: Array(0)}
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Context Before Update: (0) []
Context After Update: (1) [{…}]
Contexto alterado, enviando para a API.
Context After Update: 452a9a2ba1b14b3d4d970d5fa931591ceb4cc544dd52080e17617d414c6b8de8
Context After Update: (0) []
Prompt enviado para a API: (5) [{…}, {…}, {…}, {…}, {…}]
Texto da resposta gerada: Hello! How can I assist you today?
{1331271723303309422: Array(0)}
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Mensagem não contém 'Sapius', não processando.
--- NOVA INTERAÇÃO ---
Channel ID: 1331271723303309422
Context Before Update: (0) []
Context After Update: (1) [{…}]
Contexto alterado, enviando para a API.
Context After Update: a051da90740ddad09cf2585cd624f36beb7aa697516c2e7ff3441d9875c7ce9c
Context After Update: (0) []
Prompt enviado para a API: (5) [{…}, {…}, {…}, {…}, {…}]
Texto da resposta gerada: Olá! Em que posso te ajudar hoje?
{1331271723303309422: Array(0)}
