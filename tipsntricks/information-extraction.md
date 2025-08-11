# Information extraction Tips&Tricks

## Allow model to fail (Output format)

If you force model to provide structured output with certain schema, it will provide it no matter what. 
**Best case scenario:** you will get a JSON with all fields empty, but it looks like a hack and does not provide a lot of information anyway. **Worst case scenario:** you will extract information that looks plausible but turns out to be complete non-sense upon closer look.

What you can do instead, is to provide model with a set of tools:
- `save_document(extracted data)`
- `report_error(error description)`

This will allow the model to select the tool first and then to call it. It is still doable with plain JSON however function calling seems like a more natural abstraction and inlines with LLM "API assistant" personality.

**Bonus point:** Now the model does not jump right into JSON generation, but can output some text before function call. You can prompt the model to provide some reasoning in free form text. It will help you later to debug its "behaviour".


## Provide unambiguous definition of data and where to look for it (Prompting)

Even though some field definition may sound obvious to you, it does not mean that it is the same for the model. For example:

- Seller name in invoice: Is it official name of the company or brand name?
- Amount in invoice: Is it including taxes or excluding?
- Bank account number: Is it local BAN or IBAN?
- Bank statement date range: Is it open or closed interval?

Take a look at some of the documents, try to reason yourself why you select specific value instead of others. Write it as instructions to the model and make it as specific as possible.

## Share history of manually fixed cases with the model (In context learning)

Let's say the model works okay for most of the documents, however for one specific bank/seller/client it constantly extracts incorrect data. As soon as you see that the document is from specific bank, you could redo the recognition, but this time prepend fake "user-assistant" chat history with ground truth examples for this specific bank/seller/client. 

## "Hard" validations

When model responds with extracted data, you could actually validate it. I do not mean schema validation, but check if it is consistent with itself and with data you have saved in database before. For example:

- `invoice_total = sum(line_item.amount)`
- `closing_balance = opening_balance + sum(transaction.amount)`
- `opening_balance = account.previous_bank_statement.closing_balance`

If some of them fail, you may even try to pinpoint all possible reasons for it. It is valuable information, you can share it with the model as a response and it may fix the issue in the data. From my experience (with gemini 2.5 pro) it does not force the model to "get rid of error", it really makes the model take a second look and focus a bit more on specific part of the document and resolve the issue.

It does not always work, so you still need some fallback to handle it. Sometimes documents are just invalid.

## "Soft" validations

Besides these "hard" validations you could also run "soft" validations. If they fail it does not mean that something is certainly incorrect, but it does warn the model that maybe something is wrong and it needs to take a second look.

- `invoice.currency` is too rare for this seller
- `bank_statement.date_range` is empty but with this bank it is usually not empty

## Let the model recognize ID of database entity instead of recognizing identifiers and then matching from code

If you need to match bank statement with specific bank account. You may try to extract `account_number`/`account_number_mask`/etc. However, there are so many things that could go wrong here (it could be Local BAN instead of IBAN, or it could bank number instead of account number).

It is much easier to just provide the model with list of bank accounts you have saved in the database and ask it to extract ID of this bank account instead of matching it in code.

## A word of caution regarding using third part services

In my opinion all document extraction services could be split into 3 groups

- *Generic data extraction with user provided schema* - They just do not have any advantages over using llm directly. However you loose a fair bit of flexibility, and you need to manage contracts with one more company.

- *Specific data extraction with specific schema* - These companies have some advantages. They could implement some of the validations discussed earlier, they could even have "Human in the loop" approach implemented to provide cleanest data possible. However they still lack information you could pull from your database to cross validate everything and you have no flexibility in terms of what data you want to extract.

- *Specific data extraction with specific schema and cross validation* - These implement everything discussed above and they make sense. However: 1. You share all information with these companies, it may be too much. 2. Still no flexibility 3. I have not seen such companies yet
  
