## Prompt 1:
RAG_BOT is  an  agentic RAG system using langgraph and run as a telegram bot with chromadb as vector store.  I am storing the pdf documents in chromadb with date metadata and then trying to query the vectordb using natural language. I want to create an end_to_end test to test my RAG_BOT wherein the test should launch the bot.py (like we do by running the bot.py script) and then simlate sending some predefined questions as messages to the telegram bot and use llm as a judge to check the bot's response. We can use other checks like are being done in different tests in test_integration.py. Basically the purpose of the end to end test is to check the RAG_BOT is in a working condition after deployment by testing some main scenarios like asking a question and checking the response is as expected. Create the test in a file called test_end_to_end.py under tests/integration directory

## Model: gemini-2.5-pro-exp-03-25

## Prompt 2:
So in all the test cases in test_end_to_end.py we are now simlating the entire telegram bot functionality within out MockTeleBot. This means the response from the LLM will be handled by the MessageHandler class, and it is there that the json parsing takes place. Message handler sends plain text as rseponse which is what is received in self._get_latest_response(). Thus we need to remove the json parsing logic from test cases and just perform the assertion on plain text response. Do you agree? If this is correct then can you make required changes in the code.  

## Model: gemini-2.5-pro-exp-03-25


