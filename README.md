import configparser
import json
import asyncio
from datetime import date, datetime
import re
import numpy


from telethon import TelegramClient
from telethon.errors import SessionPasswordNeededError
from telethon.tl.functions.messages import (GetHistoryRequest)
from telethon.tl.types import (PeerChannel)

def checkForCountriesInMessage(messageText):

    countryLink=[]
    #find flags in message
    try:
        unicodes=re.split(r"\\",re.sub(r"[\x00-\x7f]+", "", messageText).encode('unicode-escape').decode('ASCII').upper()) #get all unicodes
        flagsInUnicode=[]
        for unicode in unicodes:
            if  "U0001F1E0"<=unicode<="U0001F1FF": #check if unicode is flag unicode part
                flagsInUnicode.append(unicode)        
        flagsInMessage=[ ' '.join(x) for x in zip(flagsInUnicode[0::2], flagsInUnicode[1::2]) ]#flag unicode consits of two unicodes
   
        for country in countriesList:
            if  country[0] in flagsInMessage:
                countryLink.append(country[1])
    except:
        pass


    #find country in message text
    messageWords=list(re.sub(' +', ' ',re.sub(r'[^\w]', ' ', messageText)).strip().split(' '))
    for word in messageWords:
        for idx, country in enumerate(countriesList):
            if word.lower() in list(map(str.lower,country[3::])):
                countryLink.append(country[1])
    return numpy.unique(countryLink)

# some functions to parse json date
class DateTimeEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, datetime):
            return o.isoformat()

        if isinstance(o, bytes):
            return list(o)

        return json.JSONEncoder.default(self, o)
# open country list
with open('countryList.txt','r') as countriesFile:
    lines = countriesFile.readlines()
    countriesList=[]
    for line in lines:
        countriesList.append(list(line.rstrip().split(',')))

with open('TelegramChannels.txt','r') as channelFile:
    lines = channelFile.readlines()
    telegramChannelList=[]
    for line in lines:
        telegramChannelList.append(line.strip())
# Reading Configs
config = configparser.ConfigParser()
config.read("config.ini")

# Setting configuration values
api_id = config['Telegram']['api_id']
api_hash = config['Telegram']['api_hash']

api_hash = str(api_hash)

phone = config['Telegram']['phone']
username = config['Telegram']['username']

# Create the client and connect
client = TelegramClient(username, api_id, api_hash)

for telegramChannel in telegramChannelList:
    if telegramChannel.isdigit():
        entity = PeerChannel(int(telegramChannel))
    else:
        entity = telegramChannel
    async def main(phone):
        await client.start()
        # Ensure you're authorized
        if await client.is_user_authorized() == False:
            await client.send_code_request(phone)
            try:
                await client.sign_in(phone, input('Enter the code: '))
            except SessionPasswordNeededError:
                await client.sign_in(password=input('Password: '))

        me = await client.get_me()
   

        try:
            my_channel = await client.get_entity(entity)
        except:
            print("Channel ",telegramChannel," not found" )
            return

        offset_id = 0
        limit = 100
        all_messages = []
        total_messages = 0
        total_count_limit = 0
        print("telegram channel: ", telegramChannel)
        while True:       
            history = await client(GetHistoryRequest(
                peer=my_channel,
                offset_id=offset_id,
                offset_date=None,
                add_offset=0,
                limit=limit,
                max_id=0,
                min_id=0,
                hash=0
            ))
            if not history.messages:
                break
            messages = history.messages
            for message in messages:
                message_dict=message.to_dict()
                if message_dict['_']!='Message': #some IDs are not messages
                    continue
                if message_dict['message']=='': #some IDs do not exists, maybe because the chanel admin has deleted a message
                    continue
                message_dict['CountryLink'] =checkForCountriesInMessage(message_dict['message'])
                message_dict['DownloadDateTime'] = datetime.now() 
                all_messages.append(message_dict)
            offset_id = messages[len(messages) - 1].id
            total_messages = len(all_messages)
            print("Download Status: " , total_messages, " of " ,all_messages[0].get('id'), "downloaded")
            if total_count_limit != 0 and total_messages >= total_count_limit:
                break

        with open('channel_messages.json', 'w') as outfile:
            json.dump(all_messages, outfile, cls=DateTimeEncoder)
    with client:
        client.loop.run_until_complete(main(phone))


