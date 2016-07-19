# chatlab
A place to store our adventure in self--hosted chats and disasters

**ChatLab** is the super boring name I found to identify 
the project of setting-up a software and hardware solution 
aimed to control quite a complex place: 
* a [Fabrication Laboratory](https://en.wikipedia.org/wiki/Fab_lab) (full of different machines, with specific access rights and insurance layers
* an apartment (more about this later)
* an office

Now, this document is (hopefully) going to be edited *several times* to meet several needs, which are:
- hosting a chat, with different channels, *à la slack*
- control the access to the building, from a fablab member, for a professional, from somebody (rasidency, traveller) staying in the house 
- <del>sorta of</del> adding some surveillance flavour to the place (both active and passive, bots sensors and cameras)
- hosting a internal video-com for the apartment and the 
- add more...

**Proposals:**
- [Rocket.chat](https://rocket.chat) is meeting most of our needs
- It goes even on a [Raspberry PI](https://github.com/RocketChat/Rocket.Chat.RaspberryPi)
- If hosted on a proper server (not an rpi) it has [this interesting live-chat service](https://rocket.chat/docs/administrator-guides/livechat/) to put a customer / user in contact with a specific channel 
- is [(hu)bot](https://github.com/RocketChat/hubot-rocketchat)-ready, so we won't have to migrate several telegram groups [we can hook some on a specific channel](https://rocket.chat/docs/administrator-guides/integrations/telegram)
- we can merge this user database with the one from [labAdmin project](https://github.com/FablabTorino/LabAdmin)
- we can have a basic (and role specific) surveillance infrastructure on [specific channels](https://rocket.chat/docs/user-guides/remote-video-monitoring/)
