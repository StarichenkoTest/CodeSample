# CodeSample
using System;
using System.IO;
using System.Linq;
using System.Net;
using System.Text;
using System.Web.Mvc;
using SmartSms.Models;

namespace SmartSms.Controllers
{
    public class TransactController : Controller
    {
        // GET: Transact
        [Route("")]
        [Route("Transact/")]
        [Route("Transact/Index")]
        public ActionResult Index()
        {
            return View();
        }

        //http://localhost:6372/Transact/BeelineSms?idClient=1&idCampaign=1&password=3151020&uid=bfbcde5b-7189-4044-b87d-aeeab320b288&rnd=qqq
        public ActionResult BeelineSms(int idClient, int idCampaign, string password, string uid, string rnd,
            string refURL = "")
        {
            try
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = uid;
                log.Rnd = rnd;
                log.Description = refURL;

                var res = UIdCheck(log, uid);
                if (res != "Ok") return Content(res);

                res = NumberExistanceCheck(log, uid);
                if (res != "Ok") return Content(res);
                var number = db.NumbersCode.First(x => x.Code == uid).Number;

                res = ClientExistenceCheck(log, idClient);
                if (res != "Ok") return Content(res);

                res = CampaignExistenceCheck(log, idCampaign, idClient);
                if (res != "Ok") return Content(res);
                var campaign = db.Campaigns.First(x => x.IdCampaign == idCampaign && x.IdClient == idClient);

                if (refURL != "")
                {
                    res = RefURLCheck(log, campaign, refURL);
                    if (res != "Ok") return Content(res);
                }

                res = CampaignPasswordCheck(log, campaign, password);
                if (res != "Ok") return Content(res);

                res = CampaignTypeCheck(log, campaign);
                if (res != "Ok") return Content(res);

                res = CampaignSendingTimeCheck(log, campaign);
                if (res != "Ok") return Content(res);

                res = CampaignSMSLimitCheck(log, campaign);
                if (res != "Ok") return Content(res);

                res = NumberInStopListCheck(log, idClient, idCampaign, number);
                if (res != "Ok") return Content(res);

                res = SMSToNumberTimePassedCheck(log, campaign, idClient, idCampaign, number);
                if (res != "Ok") return Content(res);

                res = CampaignDaySMSLimitCheck(log, campaign);
                if (res != "Ok") return Content(res);

                db = new WebSMSEntities1();
                var sendSMS = db.SendSMS.Create();
                sendSMS.CreationDate = DateTime.Now;
                sendSMS.IdClient = idClient;
                sendSMS.IdCampaign = idCampaign;
                sendSMS.Number = number;
                sendSMS.Status = campaign.WaitTime == 0 ? 2 : 1;
                sendSMS.Test = false;
                sendSMS.MessageText = campaign.MessageText;
                res = AddPromocode(log, campaign, sendSMS);
                if (res != "Ok") return Content(res);

                db.SendSMS.Add(sendSMS);
                db.SaveChanges();

                if (campaign.RedirectUrl != null)
                {
                    string response = SendSMS(campaign.RedirectUrl, campaign.IdClient, campaign.IdCampaign, number);
                    log.Description = response;
                }

                log.Status = 11;
                db.Log.Add(log);
                db.SaveChanges();
                return Content("SMS sended");
            }
            catch (Exception ex)
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = uid;
                log.Rnd = rnd;
                log.Status = 7;
                log.Description = $"{ex.Message}\n{ex.StackTrace}";
                db.Log.Add(log);
                db.SaveChanges();
                return Content("Unexpected Error");
            }
        }

        //http://websmsmassmess.azurewebsites.net/Transact/DirectSms?idClient=2&idCampaign=3&password=LjvLtymub&number=79652313950
        public ActionResult DirectSMS(int idClient, int idCampaign, string password, long number)
        {
            try
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = number.ToString();
                log.Rnd = "";

                var res = ClientExistenceCheck(log, idClient);
                if (res != "Ok") return Content(res);

                res = CampaignExistenceCheck(log, idCampaign, idClient);
                if (res != "Ok") return Content(res);
                var campaign = db.Campaigns.First(x => x.IdCampaign == idCampaign && x.IdClient == idClient);

                res = CampaignPasswordCheck(log, campaign, password);
                if (res != "Ok") return Content(res);

                res = CampaignTypeCheck(log, campaign);
                if (res != "Ok") return Content(res);

                db = new WebSMSEntities1();
                var sendSMS = db.SendSMS.Create();
                sendSMS.CreationDate = DateTime.Now;
                sendSMS.IdClient = idClient;
                sendSMS.IdCampaign = idCampaign;
                sendSMS.Number = number;
                sendSMS.Status = campaign.WaitTime == 0 ? 2 : 1;
                sendSMS.Test = false;
                sendSMS.MessageText = campaign.MessageText;
                res = AddPromocode(log, campaign, sendSMS);
                if (res != "Ok") return Content(res);

                db.SendSMS.Add(sendSMS);
                log.Status = 11;
                db.Log.Add(log);
                db.SaveChanges();
                return Content("SMS sended");
            }
            catch (Exception ex)
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = number.ToString();
                log.Rnd = "";
                log.Status = 7;
                log.Description = $"{ex.Message}\n{ex.StackTrace}";
                db.Log.Add(log);
                db.SaveChanges();
                return Content("Unexpected Error");
            }
        }

        //http://localhost:6372/Transact/CancellSms?idCampaign=1&password=3151020&uid=bfbcde5b-7189-4044-b87d-aeeab320b288&rnd=qqq
        public ActionResult CancellSms(int idClient, int idCampaign, string password, string uid, string rnd)
        {
            try
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = uid;
                log.Rnd = rnd;

                var res = UIdCheck(log, uid);
                if (res != "Ok") return Content(res);

                res = NumberExistanceCheck(log, uid);
                if (res != "Ok") return Content(res);
                var number = db.NumbersCode.First(x => x.Code == uid).Number;

                res = ClientExistenceCheck(log, idClient);
                if (res != "Ok") return Content(res);

                res = CampaignExistenceCheck(log, idCampaign, idClient);
                if (res != "Ok") return Content(res);
                var campaign = db.Campaigns.First(x => x.IdCampaign == idCampaign && x.IdClient == idClient);

                res = CampaignPasswordCheck(log, campaign, password);
                if (res != "Ok") return Content(res);

                var time = DateTime.Now.AddMinutes(-campaign.WaitTime);
                var record =
                    db.SendSMS.FirstOrDefault(
                        x => x.Number == number && x.Status == 1 && x.IdCampaign == idCampaign && x.CreationDate > time);
                if (record != null)
                {
                    res = RemovePromocode(log, campaign, record);
                    if (res != "Ok") return Content(res);
                    record.Status = 4;
                    log.Status = 12;
                    db.Log.Add(log);
                    db.SaveChanges();
                    return Content("SMS cancelled");
                }
                log.Status = 13;
                db.Log.Add(log);
                db.SaveChanges();
                return Content("No sms to cancell");
            }
            catch (Exception ex)
            {
                var db = new WebSMSEntities1();
                var log = db.Log.Create();
                log.CreationDate = DateTime.Now;
                log.IdClient = idClient;
                log.IdCampaign = idCampaign;
                log.Number = uid;
                log.Rnd = rnd;
                log.Status = 7;
                log.Description = $"{ex.Message}\n{ex.StackTrace}";
                db.Log.Add(log);
                db.SaveChanges();
                return Content("Unexpected Error");
            }
        }

        //public ActionResult GetSMS(int idClient, int idCampaign, string number)
        //{
        //    return Content("Ok");
        //}

        public string SendSMS(string url, long idClient, long idCampaign, long number)
        {
            try
            {
                HttpWebRequest req =
                    (HttpWebRequest)
                        WebRequest.Create(
                            $"{url}?idClient={idClient}&idCampaign={idCampaign}&number={number}");
                WebResponse resp = req.GetResponse();
                Stream stream = resp.GetResponseStream();
                if (stream != null)
                {
                    StreamReader readStream = new StreamReader(stream, Encoding.UTF8);
                    string response = readStream.ReadToEnd();
                    return response;
                }
                return null;
            }
            catch (Exception)
            {
                return "error";
            }
        }

        private string UIdCheck(Log log, string uid)
        {
            if (!uid.Equals("....................")) return "Ok";
            log.Status = 9;
            var db = new WebSMSEntities1();
            db.Log.Add(log);
            db.SaveChanges();
            return "False! Not mobile or WiFi";
        }

        private string NumberExistanceCheck(Log log, string uid)
        {
            var db = new WebSMSEntities1();
            if (db.NumbersCode.Any(x => x.Code == uid)) return "Ok";
            log.Status = 10;
            db.Log.Add(log);
            db.SaveChanges();
            return "No number for code";
        }

        private string ClientExistenceCheck(Log log, int idClient)
        {
            var db = new WebSMSEntities1();
            if (db.Clients.Any(x => x.Id == idClient)) return "Ok";
            log.Status = 14;
            db.Log.Add(log);
            db.SaveChanges();
            return "No such client";
        }

        private string CampaignExistenceCheck(Log log, int idCampaign, int idClient)
        {
            var db = new WebSMSEntities1();
            if (db.Campaigns.Any(x => x.IdCampaign == idCampaign && x.IdClient == idClient)) return "Ok";
            log.Status = 3;
            db.Log.Add(log);
            db.SaveChanges();
            return "No such campaign";
        }

        private string CampaignPasswordCheck(Log log, Campaigns campaign, string password)
        {
            var db = new WebSMSEntities1();
            if (campaign.Password == password) return "Ok";
            log.Status = 2;
            db.Log.Add(log);
            db.SaveChanges();
            return "Wrong Password";
        }

        private string CampaignTypeCheck(Log log, Campaigns campaign)
        {
            var db = new WebSMSEntities1();
            if (campaign.Type != 0) return "Ok";
            log.Status = 8;
            db.Log.Add(log);
            db.SaveChanges();
            return "Campaign is blocked";
        }

        private string CampaignSendingTimeCheck(Log log, Campaigns campaign)
        {
            var db = new WebSMSEntities1();
            if (campaign.DateStart == null || campaign.DateEnd == null ||
                (!(DateTime.Now < campaign.DateStart) && !(DateTime.Now > campaign.DateEnd))) return "Ok";
            log.Status = 6;
            db.Log.Add(log);
            db.SaveChanges();
            return "Sending time has expired";
        }

        private string CampaignSMSLimitCheck(Log log, Campaigns campaign)
        {
            var db = new WebSMSEntities1();
            if (!(db.SendSMS.Count(x => x.IdCampaign == campaign.Id) >= campaign.Limit)) return "Ok";
            log.Status = 5;
            db.Log.Add(log);
            db.SaveChanges();
            return "SMS limit is exhausted";
        }

        private string NumberInStopListCheck(Log log, int idClient, int idCampaign, long number)
        {
            var db = new WebSMSEntities1();
            if (db.WhiteList.Any(x =>
                ((x.IdClient == null || x.IdClient == idClient) && x.IdCampaign == null ||
                 x.IdClient == idClient && x.IdCampaign == idCampaign) && x.Number == number))
                return "Ok";
            if (!db.StopList.Any(
                x =>
                    ((x.IdClient == null || x.IdClient == idClient) && x.IdCampaign == null ||
                     x.IdClient == idClient && x.IdCampaign == idCampaign) && x.Number == number))
                return "Ok";
            log.Status = 1;
            db.Log.Add(log);
            db.SaveChanges();
            return "Number in StopList";
        }

        private string SMSToNumberTimePassedCheck(Log log, Campaigns campaign, int idClient, int idCampaign, long number)
        {
            var db = new WebSMSEntities1();
            var spanDate = DateTime.MinValue;
            if (campaign.SpanTime != null)
                spanDate = DateTime.Now.AddDays((double) -campaign.SpanTime);
            if (campaign.SpanTime != null &&
                !db.WhiteList.Any(x =>
                        ((x.IdClient == null || x.IdClient == idClient) && x.IdCampaign == null ||
                         x.IdClient == idClient && x.IdCampaign == idCampaign) && x.Number == number) &&
                db.SendSMS.Where(
                    x =>
                        x.CreationDate > spanDate &&
                        x.IdClient == idClient && x.IdCampaign == idCampaign).Any(x => x.Number == number))
            {
                log.Status = 4;
                db.Log.Add(log);
                db.SaveChanges();
                return "Not enough time has passed since the last sent to this number";
            }

            var min5 = DateTime.Now.AddMinutes(-5);
            if (db.WhiteList.Any(x => x.IdCampaign == idCampaign && x.Number == number) || !db.SendSMS.Where(
                x =>
                    x.CreationDate > min5 &&
                    x.IdClient == idClient && x.IdCampaign == idCampaign).Any(x => x.Number == number)) return "Ok";
            log.Status = 4;
            db.Log.Add(log);
            db.SaveChanges();
            return "Not enough time has passed since the last sent to this number";
        }

        private string CampaignDaySMSLimitCheck(Log log, Campaigns campaign)
        {
            var db = new WebSMSEntities1();
            if (!(db.SendSMS.Count(x => x.IdCampaign == campaign.Id && x.CreationDate > DateTime.Today) >=
                  campaign.DayLimit)) return "Ok";
            log.Status = 0;
            db.Log.Add(log);
            db.SaveChanges();
            return "Too many SMS";
        }

        private string RefURLCheck(Log log, Campaigns campaign, string refURL)
        {

            var db = new WebSMSEntities1();
            if (campaign.ValidateUrl == refURL) return "Ok";
            log.Status = 15;
            db.Log.Add(log);
            db.SaveChanges();
            return "Wrong query";
        }

        private string AddPromocode(Log log, Campaigns campaigns, SendSMS sendSMS)
        {
            var db = new WebSMSEntities1();
            if (!campaigns.MessageText.Contains('#')) return "Ok";
            if (
                db.PromoCodes.Any(
                    x => x.IdCampaign == campaigns.IdCampaign && x.IdClient == campaigns.IdClient && !x.IsUsed))
            {
                var promoCodeRecord =
                    db.PromoCodes.First(
                        x => x.IdCampaign == campaigns.IdCampaign && x.IdClient == campaigns.IdClient && !x.IsUsed);
                if (promoCodeRecord.PromoCode.Length != campaigns.MessageText.Count(x => x == '#'))
                {
                    log.Status = 16;
                    db.Log.Add(log);
                    db.SaveChanges();
                    return "Wrong promocode";
                }
                promoCodeRecord.IsUsed = true;
                string hash = new string('#', promoCodeRecord.PromoCode.Length);
                sendSMS.MessageText = campaigns.MessageText.Replace(hash, promoCodeRecord.PromoCode);
                db.SaveChanges();
                return "Ok";
            }
            log.Status = 17;
            db.Log.Add(log);
            db.SaveChanges();
            return "Promocodes have ended";
        }

        private string RemovePromocode(Log log, Campaigns campaigns, SendSMS sendSMS)
        {
            var db = new WebSMSEntities1();
            if (!campaigns.MessageText.Contains('#')) return "Ok";
            int length = campaigns.MessageText.Count(x => x == '#');
            string promocode = sendSMS.MessageText.Substring(campaigns.MessageText.IndexOf('#'), length);

            if (db.PromoCodes.Any(
                x =>
                    x.IdCampaign == campaigns.IdCampaign && x.IdClient == campaigns.IdClient &&
                    x.PromoCode == promocode && x.IsUsed))
            {
                var promoCodeRecord =
                    db.PromoCodes.First(
                        x =>
                            x.IdCampaign == campaigns.IdCampaign && x.IdClient == campaigns.IdClient &&
                            x.PromoCode == promocode && x.IsUsed);

                promoCodeRecord.IsUsed = false;
                db.SaveChanges();
                return "Ok";
            }
            return "Ok";
        }
    }
}
