using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.Serialization;
using System.ServiceModel;
using System.Text;
using System.Data.Entity;


namespace Server
{
    [ServiceBehavior(InstanceContextMode = InstanceContextMode.Single,
        ConcurrencyMode =ConcurrencyMode.Multiple)]
    public class Server : IServer
    {
       // private DataBase baseofData = new DataBase();
        private List<Abonent> allAbonents =  new List<Abonent>();
        private int idAbonent = 0;




        private void PushMessage(int senderId,int recipientId,string textOfMessage)
        {
            using (var context = new DataBase()) 
            {
                var message = new MessageDb()
                {
                    SenderId = senderId,
                    RecipientId = recipientId,
                    TextOfMessage = textOfMessage
                };

                context.Messages.Add(message);
                context.SaveChanges();
            }
        }


        private List<MessageDb> PopMessage(int recipientId)
        {
            using (var context = new DataBase())
            {
                var messagesInDb = context.Messages.Where(i => (i.RecipientId == recipientId));


                if (messagesInDb != null)
                {
                    foreach (var message in messagesInDb)
                        context.Messages.Remove(message);
                    return messagesInDb.ToList();
                }
                else
                {
                    return null;
                }
               
            }

            
        }


        public void SendMessage(int senderId, string[] recipientNames, string message)
        {
            Abonent sender = allAbonents.Find(ab => ab.ID == senderId);
            Console.WriteLine(sender.Name + " отправляет всем сообщение");
            if (recipientNames == null) //оправить всем
            {
                foreach (Abonent index in allAbonents)
                {
                    if (index.Status) 
                    {
                        index.Callback.cbSendMessage(sender.Name, message);
                    }
                    else
                    {
                        PushMessage(sender.ID, index.ID, message);
                    }
                    
                }
                Console.WriteLine(sender.Name + " отправил всем сообщение");
            }
            else
            {
                foreach (string index in recipientNames)
                {
                    Abonent recipient = allAbonents.Find(ab => ab.Name == index);
                    if (recipient.Status)
                    {
                        recipient.Callback.cbSendMessage(sender.Name, message);
                    }
                    else
                    {

                        PushMessage(senderId, recipient.ID, message); //сохранить сообщение в базу данных
                    }
                }
            }
           
           
        }
        public void ShowAbonents(int id)
        {
            Abonent abonent = allAbonents.Find(ab => ab.ID == id); //мб быстрее будет вызвать напрямую  OperationContext.Current.GetCallbackChannel<IMessageCallback>() ?
            foreach (Abonent index in allAbonents) // и стоит ли вообще вызывать столько callbackов, мб стоит добавить еще одну функцию cbShowAbonents(string[]names ,string[] status)
            {
               if (index.ID != id)
               {
                   abonent.Callback.cbShowAbonent(index.Name, index.Status);
               }
            }
        }

        //private void ProvideMessage(int id) // мб его все таки включить в RPC и вызывать отдельно клиентом,чтобы не захламлять Connect
        //{
            
        //    Abonent   = allAbonents.Find(ab => ab.ID == id);
        //    //if (abonent.IsNewMessage)
        //    {

        //        var messagesInDb = PopMessage(abonent.ID); //взять сообщение из базы данных
        //        //    
        //        foreach (MessageDb message in messagesInDb)
        //        {
        //            abonent.Callback.cbSendMessage(abonent.Name, message.TextOfMessage);
        //        }
        //        abonent.IsNewMessage = false;
        //    }

            
       // }
        public int Connect(string name)
        {
            Abonent abonent;
            string str;
            if (allAbonents.Exists(ab => ab.Name == name))
            {
                str = "существующий ";
                abonent = allAbonents.Find(ab => ab.Name == name);
                abonent.Callback = OperationContext.Current.GetCallbackChannel<IMessageCallback>();
                abonent.Status = true;
            }
            else
            {
                str = "новый ";
                abonent = new Abonent(idAbonent++, name, OperationContext.Current.GetCallbackChannel<IMessageCallback>());
                allAbonents.Add(abonent);
            }

            //Дать знать остальным пользователям о подключении нового
            foreach (Abonent index in allAbonents)
            {
                if (index.Status && index.ID != abonent.ID)
                {
                    index.Callback.cbShowAbonent(abonent.Name, true);
                }
            }

         //   ProvideMessage(abonent.ID); //предоставить пользователю непринятые сообщения
            Console.WriteLine("Подключился " + str + abonent.Name);
            return abonent.ID;

        }
        public void Disconnect(int id) 
        {
            Abonent abonent = allAbonents.Find(ab => ab.ID == id);
            abonent.Status = false;
            abonent.Callback = null;
        }
    }
}
