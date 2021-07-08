#!/usr/bin/python3.6
# -*- coding: utf-8 -*-
#  author: Jervis

import json
import requests
import datetime
import pymysql
import logging
import traceback
from requests.exceptions import RequestException
from config import *
import sys
import os

url = 'https://eeir.sipg.com.cn/EIRIMS/servlet/WebRpcServlet'
user_agent = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/76.0.3809.100 Saf'

# 待接单
not_received = 'O1'
# 已接单
is_received = 'O2'
# 已派单待确认
not_confirm = 'O3'
# 司机已接单
driver_received = 'O4'
# 已转车队
is_transfer = 'O5'
# 正常核销
normal_over = 'XN'
# 异常核销
abnormal_over = 'XE'


log_path = '/opt/data/logs/eir_log'
if os.path.exists(log_path):
    pass
else:
    os.makedirs(log_path)
logging.basicConfig(filename=log_path + '/eir_spider_' + str(datetime.date.today().year) + ('%02d' % datetime.date.today().month) + '.log',
                    format='%(asctime)s %(filename)s[line:%(lineno)d] %(levelname)s %(message)s',
                    level=logging.INFO, filemode='a', datefmt='[%Y-%m-%d %H:%M:%S]')


# 发送请求操作
def requests_post(request_url, data, headers):
    try:
        resp = requests.post(url=request_url, data=data, headers=headers)
        if resp.status_code == 200:
            logging.info("请求成功！")
            return resp
        return None
    except RequestException as re:
        logging.error("请求失败！！Error：\n" + traceback.format_exc())
        return None


# login操作
def login():
    logging.info("开始登陆操作！")
    yzm_data = {"fromSource": "", "busName": "Img", "busFunc": "VerificationCode", "busParams": {}}
    yzm_headers = {'Content-Type': 'application/json',
               'Referer': 'https://eeir.sipg.com.cn/signIn.html?v=6d860d14d4',
               'User-Agent': user_agent}
    # 获取验证码信息
    logging.info("发送请求，获取验证码...")
    yzm_res = requests_post(url, json.dumps(yzm_data), yzm_headers)
    # 把返回的结果转化成Json
    json_yzm = json.loads(yzm_res.text)
    # 提取验证码code
    yzm_code = json.loads(json_yzm['resultSet'])['code']
    # 获取session
    yzm_session = yzm_res.cookies.__getitem__('JSESSIONID')

    # login用的request data
    login_data = {"fromSource": "",
            "busName": "LoginBusiness",
            "busFunc": "login",
            "busParams":
                {"userName": EIR_USER_NAME,
                 "Password": EIR_PASSWORD,
                 "yzm": yzm_code}
            }

    login_cookie = 'JSESSIONID=' + yzm_session
    login_headers = {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
               'Referer': 'https://eeir.sipg.com.cn/signIn.html?v=6d860d14d4',
               'User-Agent': user_agent,
               'Cookie': login_cookie}
    # 登陆
    logging.info("发送请求，进行模拟登陆")
    requests_post(url, json.dumps(login_data), login_headers)

    return yzm_session


# 获取EIR追踪信息
def get_eir_status(login_session, receipt_id):
    eir_status_data = {"busName": "PublicQuery", "busFunc": "EirStatusList", "busParams": {"receiptId": 0000000}}
    # 设置eir详细ID
    eir_status_data['busParams']['receiptId'] = receipt_id
    # 设置cookie
    eir_status_cookie = 'JSESSIONID=' + login_session
    eir_status_headers = {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
               'Referer': 'https://eeir.sipg.com.cn/transport-r_m.html?v=ce87ea990b',
               'User-Agent': user_agent,
               'Cookie': eir_status_cookie}
    # 发送获取eir列表信息的请求
    logging.info("发送请求，获取EIR追踪信息。EIR详情ID：" + str(receipt_id))
    eir_status_res = requests_post(url, json.dumps(eir_status_data), eir_status_headers)
    return eir_status_res.text


# 发送eir请求
def send_eir_request(data, headers):
    res = requests_post(url, json.dumps(data), headers)
    dic_res = json.loads(res.text)
    list_result = []
    # 如果返回的Json数据中包含totalRow，并且totalRow > 0的时候，则解析返回的Json数据
    if 'resultSet' in dic_res.keys() and 'totalRow' in json.loads(dic_res['resultSet']).keys() \
            and int(json.loads(dic_res['resultSet'])['totalRow']) > 0:
        logging.info("取到 " + str(json.loads(dic_res['resultSet'])['totalRow']) + " 件数据")
        # 循环遍历取出的【未接单】数据
        list_result = json.loads(dic_res['resultSet'])['CtnList']
        total_page = int(json.loads(dic_res['resultSet'])['totalPage'])
        if total_page > 1:
            logging.info("数据大于1页，总页数：" + str(total_page))
            list_result = []
            a = 1
            while a <= total_page:
                data['busParams']['pageIndex'] = a
                logging.info("发送请求，获取第" + str(a) + "页数据")
                res = requests_post(url, json.dumps(data), headers)
                dic_res = json.loads(res.text)
                # 拼接list
                list_result = list_result + json.loads(dic_res['resultSet'])['CtnList']
                a += 1
    else:
        logging.info("没有取到的数据！")

    return list_result


# 获取EIR列表信息
def get_eir_list(login_session, current_date, db_config):
    data = {"fromSource": "",
        "busName": "QueryEIR",
        "busFunc": "EirChuanQuery",
        "busParams": {"pageIndex": 1,
            "pageSize": 40,
            "queryType": 3,
            "asc": False,
            "orderStr": "receiptDate",
            "category": "O1",
            "receiptDate": "2019-08-19",
            "receiptEndDate": "2019-08-19",
            "trustId": "",
            "vesselEname": "",
            "voyageNo": "",
            "billNo": "",
            "cntrNo": "",
            "putCstName": "",
            "transCstName": "",
            "delivPlaceStr": "",
            "rtnPlaceStr": "",
            "receiptDateTime": "",
            "receiptEndDateTime": "",
            "driverName": "",
            "truckNo": "",
            "planNo": "",
            "reservedNo": "",
            "delivPlaceName": "",
            "rtnPlaceName": "",
            "ownerCstName": "",
            "shiperMemo": "",
            "leaseCstName": "",
            "loadPortCode": "",
            "dischrgPortCode": "",
            "destPortCode": "",
            "transPortCode": "",
            "queryNo": ""}
        }

    # 设定起始时间参数为昨天日期，从而获取昨天和今天天数据
    one_day = datetime.timedelta(days=1)
    two_day = datetime.timedelta(days=2)
    # 1天前的日期
    one_day_before = str(current_date - two_day)
    current_date = str(current_date - one_day)

    data['busParams']['receiptDate'] = one_day_before
    data['busParams']['receiptEndDate'] = current_date

    # 设置cookie
    cookie = 'JSESSIONID=' + login_session
    headers = {'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8',
               'Referer': 'https://eeir.sipg.com.cn/transport-eir_s.html?v=ce87ea990b',
               'User-Agent': user_agent,
               'Cookie': cookie}

    # 设定接单状态参数为【未接单】
    data['busParams']['category'] = not_received
    data['busParams']['pageIndex'] = 1
    # 发送获取【未接单】eir列表信息的请求
    logging.info("===发送请求，开始获取【未接单】eir列表信息===")
    list_not_receive = send_eir_request(data, headers)
    # for index in range(len(list_not_receive)):
    #     print(list_not_receive[index]['receiptNo'])

    # 设定接单状态参数为【已接单】
    data['busParams']['category'] = is_received
    data['busParams']['pageIndex'] = 1
    # 发送获取【已接单】eir列表信息的请求
    logging.info("发送请求，开始获取【已接单】eir列表信息")
    list_is_received = send_eir_request(data, headers)

    # 设定接单状态参数为【已派单待确认】
    data['busParams']['category'] = not_confirm
    data['busParams']['pageIndex'] = 1
    # 发送获取【已接单】eir列表信息的请求
    logging.info("===发送请求，开始获取【已派单待确认】eir列表信息===")
    list_not_confirm = send_eir_request(data, headers)

    # 设定接单状态参数为【司机已接单】
    data['busParams']['category'] = driver_received
    data['busParams']['pageIndex'] = 1
    # 发送获取【司机已接单】eir列表信息的请求
    logging.info("===发送请求，开始获取【司机已接单】eir列表信息===")
    list_driver_received = send_eir_request(data, headers)

    # 设定接单状态参数为【已转车队】
    data['busParams']['category'] = is_transfer
    data['busParams']['pageIndex'] = 1
    # 发送获取【已转车队】eir列表信息的请求
    logging.info("===发送请求，开始获取【已转车队】eir列表信息===")
    list_is_transfer = send_eir_request(data, headers)

    # 设定接单状态参数为【正常核销】
    data['busParams']['category'] = normal_over
    data['busParams']['pageIndex'] = 1
    # 发送获取【正常核销】eir列表信息的请求
    logging.info("===发送请求，开始获取【正常核销】eir列表信息===")
    list_normal_over = send_eir_request(data, headers)

    # 设定接单状态参数为【异常核销】
    data['busParams']['category'] = abnormal_over
    data['busParams']['pageIndex'] = 1
    # 发送获取【异常核销】eir列表信息的请求
    logging.info("===发送请求，开始获取【异常核销】eir列表信息===")
    list_abnormal_over = send_eir_request(data, headers)

    # 合并所有接单状态的数据
    # list_eir = list_not_receive + list_is_received + list_not_confirm + list_driver_received +\
    #            list_is_transfer + list_normal_over + list_abnormal_over
    list_eir = list_abnormal_over + list_normal_over + list_is_transfer + list_driver_received + \
               list_not_confirm + list_is_received + list_not_receive
    # 循环遍历
    param_dict = {}
    logging.info("循环遍历所有接单状态的数据，作成insert语句的参数")
    for index in range(len(list_eir)):
        dic_eir = list_eir[index]

        # EIR编号
        receipt_no = ''
        if 'receiptNo' in dic_eir.keys(): receipt_no = dic_eir['receiptNo']
        # 提单号
        bill_nbr = ''
        if 'billNbr' in dic_eir.keys(): bill_nbr = dic_eir['billNbr']
        # EIR详情ID
        receipt_id = ''
        if 'receiptId' in dic_eir.keys(): receipt_id = dic_eir['receiptId']
        # 是否接单(O6:待接单, O7:已接单)
        value = ''
        if 'value' in dic_eir.keys(): value = dic_eir['value']
        # 箱型尺寸
        cntr_size_code = ''
        if 'cntrSizeCode' in dic_eir.keys(): cntr_size_code = dic_eir['cntrSizeCode']
        # 箱型代码
        cntr_type_code = ''
        if 'cntrTypeCode' in dic_eir.keys(): cntr_type_code = dic_eir['cntrTypeCode']
        # 箱号
        cntr_no = ''
        if 'cntrNo' in dic_eir.keys(): cntr_no = dic_eir['cntrNo']
        # 提箱点
        deliv_place_str_base = ''
        if 'delivPlaceStrBase' in dic_eir.keys(): deliv_place_str_base = dic_eir['delivPlaceStrBase']
        # 提箱点代码
        deliv_place_code = ''
        if 'delivPlaceCode' in dic_eir.keys(): deliv_place_code = dic_eir['delivPlaceCode']
        # 铅封号
        seal_no = ''
        if 'sealNo' in dic_eir.keys(): seal_no = dic_eir['sealNo']
        # 关联EIR编号
        rep_receipt_no = None
        if 'repReceiptNo' in dic_eir.keys(): rep_receipt_no = dic_eir['repReceiptNo']
        # 是否关联其他eir编号
        repcntr_flag = '0'
        if 'repcntrFlag' in dic_eir.keys(): repcntr_flag = dic_eir['repcntrFlag']

        receipt_date = ''
        if 'receiptDate' in dic_eir.keys(): receipt_date = dic_eir['receiptDate']
        # if len(receipt_date) > 10:
        #     receipt_date = receipt_date[0:10]

        eir_json = str(json.dumps(dic_eir, ensure_ascii=False))
        eir_status_json = ''
        # 获取追踪信息(！！因为请求量太大，所以不去定期获取追踪信息！！)
        # eir_status_json = get_eir_status(login_session, receipt_id)
        # now = datetime.datetime.now()

        param = (receipt_no, bill_nbr, receipt_id, value, cntr_size_code, cntr_type_code, cntr_no, deliv_place_str_base,
                 deliv_place_code, seal_no, rep_receipt_no, repcntr_flag, receipt_date, eir_json, eir_status_json,
                 value, cntr_size_code, cntr_type_code, cntr_no, deliv_place_str_base, deliv_place_code,
                 seal_no, rep_receipt_no, repcntr_flag, eir_json,receipt_id,receipt_date)
        # param_list.append(param)
        param_dict[receipt_no] = param

    # 将字典的所有值转换成list
    param_list = list(param_dict.values())

    conn = create_connect(db_config)

    try:
        with conn.cursor() as cursor:
            # # 先删除当天旧的数据
            # logging.info("删除当天和昨天旧的数据！")
            # del_sql = "delete from eir_spider where " \
            #           "receipt_date BETWEEN '" + one_day_before + "' and '" + current_date + "'"
            # cursor.execute(del_sql)
            # 插入新的数据
            logging.info("插入最新的数据！如果数据存在则更新")
            sql = "insert into eir_spider(" \
                  "receipt_no, bill_nbr, receipt_id, value, cntr_size_code, cntr_type_code, cntr_no, " \
                  "deliv_place_str_base, deliv_place_code, seal_no, rep_receipt_no, repcntr_flag, receipt_date, " \
                  "eir_json, eir_status_json,create_date) " \
                  "values(%s, %s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,%s,now()) ON DUPLICATE KEY UPDATE " \
                  "value = %s, cntr_size_code = %s, cntr_type_code = %s, cntr_no = %s, deliv_place_str_base = %s, " \
                  "deliv_place_code = %s, seal_no = %s, rep_receipt_no = %s, repcntr_flag = %s, eir_json = %s,receipt_id=%s ,receipt_date=%s ," \
                  "update_date = now()"
            cursor.executemany(sql, param_list)
            conn.commit()
    except Exception as e:
        logging.error("数据库操作失败，Error：\n" + traceback.format_exc())
        conn.rollback()
    finally:
        conn.close()


# 获取EIR详细信息
def get_eir_detail(login_session, receipt_id):

    eir_detail_data = {"busName": "QueryEIR", "busFunc": "GetEirDetails", "busParams": {"id": 0000000}}
    # 设置eir详细ID
    eir_detail_data['busParams']['id'] = receipt_id
    # 设置cookie
    cookie = 'JSESSIONID=' + login_session
    eir_detail_headers = {'Content-Type': 'application/json',
               'Referer': 'https://eeir.sipg.com.cn/transport-r_m.html?v=ce87ea990b',
               'User-Agent': user_agent,
               'Cookie': cookie}
    # 发送获取eir列表信息的请求
    detail_res = requests.post(url=url, data=json.dumps(eir_detail_data), headers=eir_detail_headers)
    print(detail_res.text)


# 创建数据库连接
def create_connect(db_config):
    if 'PRO' == db_config:
        host = DB_PRO_HOST
        port = DB_PRO_PORT
        user = DB_PRO_USER
        passwd = DB_PRO_PASSWD
        db = DB_PRO_DB
    else:
        host = DB_QA_HOST
        port = DB_QA_PORT
        user = DB_QA_USER
        passwd = DB_QA_PASSWD
        db = DB_QA_DB

    conn = pymysql.connect(host=host, port=port, user=user, passwd=passwd, db=db, use_unicode=True, charset='utf8')
    return conn

if __name__ == '__main__':
    logging.info("===========================处理开始=============================")
    if len(sys.argv) < 2:
        print("请输入参数（测试环境：QA，生成环境：PRO）")
        exit()
    db_config = sys.argv[1]
    logging.info("输入参数为：" + str(db_config))

    try:
        # 登陆操作
        login_session = login()
        # 取得当天日期
        # current_date = datetime.date.today().isoformat()
        current_date = datetime.date.today()
        # 获取EIR列表信息
        get_eir_list(login_session, current_date, db_config)
    except Exception:
        logging.error("System error:" + traceback.format_exc())

    logging.info("===========================处理结束=============================")
    # 获取EIR详细信息
    # receipt_id = '9719539'
    # get_eir_detail(login_session, receipt_id)
