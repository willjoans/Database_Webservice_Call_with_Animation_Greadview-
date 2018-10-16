Web_Service Call And Database Call

*********pod Install*******************
    pod 'Alamofire', '~> 4.7'
    pod 'SVProgressHUD'
    pod 'SDWebImage', '~> 4.0'
    pod 'RealmSwift'

*******Connection to Data in Database********
class Connectivity {
    class func isConnectedToInternet() ->Bool {
        return NetworkReachabilityManager()!.isReachable
    }
}

 if Connectivity.isConnectedToInternet() {
        //Api Call
 }else{
       // Connection Data not Found
 }
 
 ***********object Classs
 import UIKit
import RealmSwift
import Realm
class MovieBean: Object {
    
    
    dynamic var MovieID = 0
    dynamic var Images = ""
    dynamic var Movierating = ""
    dynamic var movieTitle = ""
    dynamic var MovieYear = ""
    
    
    override static func primaryKey() -> String? {
        return "MovieID"
    }
    func incrementID() -> Int {
        let realm = try! Realm()
        return (realm.objects(MovieBean.self).max(ofProperty: "MovieID") as Int? ?? 0) + 1
    }

}
 
************* //////ServerCalll
import Alamofire
import Foundation

private var URL_BASE = "http://api.androidhive.info/json/"
var URL_MOVIE = URL_BASE + "movies.json"

enum HTTP_METHOD {
    case http_GET, http_POST
}
enum ServerCallName : Int {
    case MovieDetaile = 101
}
protocol ServerCallDelegate {
    func ServerCallSuccess(_ resposeObject: AnyObject, name: ServerCallName)
    func ServerCallFailed(_ errorObject:String, name: ServerCallName)
}
class ServerCall: NSObject {
    var delegateObject : ServerCallDelegate!
    
    // Shared Object Creation
    static let sharedInstance = ServerCall()

// Make API Request WITH Header
func requestWithUrlAndHeader(_ httpMethod: HTTP_METHOD, urlString: String, header: [String : String], delegate: ServerCallDelegate, name: ServerCallName) {
    self.delegateObject = delegate
    let methodOfRequest : HTTPMethod = (httpMethod == HTTP_METHOD.http_GET) ? HTTPMethod.get : HTTPMethod.post
    
    let queue = DispatchQueue(label: "com.versatiletechno.manager-response-queue", attributes: DispatchQueue.Attributes.concurrent)
    
    let request = Alamofire.request(urlString, method: methodOfRequest, parameters: nil, encoding: JSONEncoding.default, headers: header)
    
    request.responseJSON(queue: queue,options: JSONSerialization.ReadingOptions.allowFragments)
    {
        (response : DataResponse<Any>)
        in DispatchQueue.main.async
            {
                print("Am I back on the main thread: \(Thread.isMainThread)")
                if (response.result.isSuccess) {
                    self.delegateObject.ServerCallSuccess(response.result.value! as AnyObject, name: name)
                }
                else if (response.result.isFailure) {
                    self.delegateObject.ServerCallFailed((response.result.error?.localizedDescription)!, name: name)
                }
        }
      }
    }
}

************ServerCall

import UIKit
import Kingfisher
import RealmSwift
import Realm
//import svprogree
class ViewController: UIViewController,UITableViewDelegate,UITableViewDataSource,ServerCallDelegate {
    
    
    
    @IBOutlet weak var tblmovieView: UITableView!
    var MovieArry = [MovieBean]()
    var MBean = MovieBean()
    let refreshcontroller = UIRefreshControl()
    let realm = try? Realm()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        refreshcontroller.addTarget(self, action: #selector(self.APICall), for: UIControlEvents.valueChanged)
        refreshcontroller.tintColor = UIColor.darkGray
        tblmovieView.addSubview(refreshcontroller)
    }
    
    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        self.APICall()
    }
    
    func APICall(){
     //   SVProgressHUD.show(withStatus: "Processing...")
        ServerCall.sharedInstance.requestWithUrlAndHeader(.http_GET, urlString: URL_MOVIE, header:["":""], delegate: self, name: .MovieDetaile)
    }
    func ServerCallSuccess(_ resposeObject: AnyObject, name: ServerCallName) {
         print("ServerCall Sucess!")
         refreshcontroller.endRefreshing()
         ResquestData(resposeObject)
    }
    
    func ServerCallFailed(_ errorObject: String, name: ServerCallName) {
        print("ServerCall Faill!")
         refreshcontroller.endRefreshing()
    }
    
    func ResquestData(_ objResponce:AnyObject){
        print(objResponce)
        let ResponceData = objResponce as! [AnyObject]
        print(ResponceData)
        self.MovieArry.removeAll()
        if let ResponArray = ResponceData as? [AnyObject]{
            
            for ArratData in ResponArray{
                if let DataResult = ArratData as? [String:Any]{
                    let Bean = MovieBean()
                    do {
                        let realm = try Realm()
                        Bean.MovieID  = Bean.incrementID()
                        Bean.Movierating = String(describing: DataResult["rating"]!)
                        Bean.MovieYear = String(describing: DataResult["releaseYear"]!)
                        Bean.movieTitle = String(describing: DataResult["title"]!)
                        Bean.Images = String(describing: DataResult["image"]!)
                        try! realm.write {
                            realm.add(Bean)
                        }
                        print(Bean)
                        print("------Sucefullay Update-------->")
                    }
                    catch let error {
                        
                        print(".........Error in add User function.........")
                        print(error.localizedDescription)
                        
                    }
                    
                    self.MovieArry.append(Bean)
                }
            }
            self.tblmovieView.reloadData()
        }
    }

    
            //MARK: tableView method
            func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
                return MovieArry.count
            }
            func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
                let cell:MovieCell = tableView.dequeueReusableCell(withIdentifier: "MovieCell")as! MovieCell
                let Bean = MovieArry[indexPath.row]
                cell.lblmoviename.text = Bean.movieTitle
                let imgUrl = URL(string: Bean.Images)
                cell.imgmovie.kf.setImage(with: imgUrl)
                cell.lblReview.text = String(Bean.Movierating)
                cell.lblyear.text = String(Bean.MovieYear)
                return cell
            }
            func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
                let Bean = MovieArry[indexPath.row]
                MBean = Bean
                self.performSegue(withIdentifier: "MovieDetileRoot", sender: self)
                
            }
            
            func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCellEditingStyle, forRowAt indexPath: IndexPath) {
                
                if editingStyle == .delete {
                    try! realm?.write {
                        realm?.delete(MovieArry[indexPath.row])
                    }
                    
                    MovieArry.remove(at: indexPath.row)
                    
                    tableView.deleteRows(at: [indexPath], with: .fade)
                }
                
            }
            override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
                if segue.identifier == "MovieDetileRoot"{
                    if let MovieDetaile = segue.destination as? MovieDetaileVC{
                        MovieDetaile.Bean = self.MBean
                    }
                }
            }
            
}
/*************************Database Data and 
TableList Data


====================BEAN===========>>>>>>>>>>>>>>> import Foundation import RealmSwift import Realm

class Student:Object { dynamic var student_id = 0 dynamic var student_Name = "" dynamic var student_Collage = "" dynamic var student_Email = "" dynamic var student_Mark = "" dynamic var student_Mobile = ""

 override static func primaryKey() -> String? {
    return "student_id"
}

func incrementID() -> Int {
    let realm = try! Realm()
    return (realm.objects(Student.self).max(ofProperty: "student_id") as Int? ?? 0) + 1
}
} ========================STUDETNLIST=========SELECQUERY======DELETE====>>>>>>>>>>>>>>>>>>>>

import UIKit import Realm import RealmSwift

class studentList: UIViewController,UITableViewDelegate,UITableViewDataSource {

//MARK:- ViewContrller outlet

@IBOutlet weak var tblviewStudentList: UITableView!


let realm = try! Realm()
var studarry : Results<Student>?
var selectedBean = Student()

//MARK:-ViewController LifeCycel

override func viewDidLoad() {
    super.viewDidLoad()
    
}
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    studarry = realm.objects(Student.self)
    print(studarry!)
    self.tblviewStudentList.reloadData()
}
//MARK:- TableVie method
func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
    return (studarry?.count)!
}
func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
    let cell:ListCell = (tableView.dequeueReusableCell(withIdentifier: "Cell")as? ListCell)!
    let Bean = self.studarry?[indexPath.row]
    cell.lblName.text = Bean?.student_Name
    cell.lblMArk.text = String(describing: Bean!.student_Mark)
    cell.lblEmail.text = Bean?.student_Email
    cell.btnadddetaile.tag = indexPath.row
    return cell
}

func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
    let story = UIStoryboard.init(name: "Main", bundle: nil)
    let StuListVC = story.instantiateViewController(withIdentifier: "ViewController")as? ViewController
    let bean = studarry?[indexPath.row]
    StuListVC?.SelectedBean = bean!
    StuListVC?.EditStudet = false
    self.navigationController!.pushViewController(StuListVC!, animated: true)
}

func tableView(_ tableView: UITableView, commit editingStyle: UITableViewCellEditingStyle, forRowAt indexPath: IndexPath) {
    if editingStyle == .delete{
        if let item = studarry?[indexPath.row] {
            try!  realm.write {
                realm.delete(item)
            }
            tableView.deleteRows(at: [indexPath], with: UITableViewRowAnimation.automatic)
     
    }
}
} //MARK:- Uibutton Action @IBAction func TapToAddDetaile(_ sender: UIButton) { let Bean = studarry?[sender.tag] self.selectedBean = Bean! print(Bean ?? "") self.performSegue(withIdentifier: "SeguaDetaileRoot", sender: self)

}
@IBAction func TapToInsertData(_ sender: Any) {
    self.performSegue(withIdentifier: "ViewListRoot", sender: self)
}


//MARK:- navigation
override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
    if segue.identifier == "SeguaDetaileRoot"{
        if let DetaileVC = segue.destination as? StudentDetaile{
            DetaileVC.Bean = selectedBean
        }
    }
}
}

===========================INSER,UPDATE======================>>>>>>>>>>>>>>>>>>>>>>>>>>>

import UIKit import RealmSwift import Realm

class ViewController: UIViewController {

// MARK:- ViewController Outlet

@IBOutlet weak var txtNAme: UITextField!
@IBOutlet weak var txtEmail: UITextField!
@IBOutlet weak var txtCollage: UITextField!
@IBOutlet weak var txtmobile: UITextField!
@IBOutlet weak var txtmark: UITextField!
var EditStudet :Bool = true
var s_id = Int()
var SelectedBean = Student()

//MARK:- ViewController LifeCycle
override func viewDidLoad() {
    super.viewDidLoad()
    
}
override func viewWillAppear(_ animated: Bool) {
    super.viewWillAppear(animated)
    
    self.txtNAme.text = SelectedBean.student_Name
    self.txtEmail.text = SelectedBean.student_Email
    self.txtmobile.text = String(SelectedBean.student_Mobile)
    self.txtmark.text = String(SelectedBean.student_Mark)
    self.txtCollage.text = SelectedBean.student_Collage
    self.s_id = SelectedBean.student_id

}

func InstrstudentData(){
    do {
        let realm = try Realm()
        let Stdeunt_Data = Student()
        Stdeunt_Data.student_id  = Stdeunt_Data.incrementID()
        Stdeunt_Data.student_Name = txtNAme.text!
        Stdeunt_Data.student_Email = txtEmail.text!
        Stdeunt_Data.student_Collage = txtCollage.text!
        Stdeunt_Data.student_Mobile = (txtmobile.text!)
        Stdeunt_Data.student_Mark = (txtmark.text!)
        
        try realm.write {
            realm.add(Stdeunt_Data)
        }
        print(Stdeunt_Data)
        print("--------Sucefullay InsertData------>")
    }
    catch let error {
        
        print(".........Error in add User function.........")
        print(error.localizedDescription)
        
    }
}

func UdateData(){
  
    do {
        let realm = try Realm()
        
        let StdeuntData = Student()
        print(s_id)
        StdeuntData.student_id  = s_id
        print(StdeuntData.student_id)
        StdeuntData.student_Name = txtNAme.text!
        StdeuntData.student_Email = txtEmail.text!
        StdeuntData.student_Collage = txtCollage.text!
        StdeuntData.student_Mobile = (txtmobile.text!)
        StdeuntData.student_Mark = (txtmark.text!)
        
        try! realm.write {
            realm.add(StdeuntData, update: true)
        }
        
        print(StdeuntData)
        print("------Sucefullay Update-------->")
    }
    catch let error {
        
        print(".........Error in add User function.........")
        print(error.localizedDescription)
        
    }
}
private func validate() -> Bool {
    var msg : String? = nil
    
    if (txtNAme.text?.isEmpty == true) {
        msg = "Please \(txtNAme.placeholder!)"
    }
    else if (txtEmail.text?.isEmpty == true) {
        msg = "Please \(txtEmail.placeholder!)"
    }
    else if (txtCollage.text?.isEmpty == true) {
        msg = "Please \(txtCollage.placeholder!)"
    }
    else if (txtmobile.text?.isEmpty == true) {
        msg = "Please \(txtmobile.placeholder!)"
    }
    else if (txtmark.text?.isEmpty == true) {
        msg = "Please \(txtmark.placeholder!)"
    }
    
    
    if msg != nil {
        let alert = UIAlertController(title:"Alert!", message: msg!, preferredStyle: UIAlertControllerStyle.alert)
        alert.addAction(UIAlertAction(title: "OK", style: UIAlertActionStyle.default, handler: nil))
        self.present(alert, animated: true, completion: nil)
        return false
    }
    
    return true
}

@IBAction func TapTOback(_ sender: Any) {
    self.navigationController!.popViewController(animated: true)
}

//MARK:- Button Action
@IBAction func TapTosubmit(_ sender: Any) {
    
    if validate() == false{
        return
    }
    if EditStudet == true{
        InstrstudentData()
        self.navigationController!.popViewController(animated: true)
    }
    else{
        self.UdateData()
        self.navigationController!.popViewController(animated: true)
    }
}
} ====================Detaile Database=========================>>>>>>>>>>>>>>>>>>>>>>>>

import UIKit

class StudentDetaile: UIViewController {

//MARK:-Viewcontrollr outlet

@IBOutlet weak var lblName: UILabel!
@IBOutlet weak var lblEmail: UILabel!
@IBOutlet weak var lblcollage: UILabel!
@IBOutlet weak var lblmobile: UILabel!
@IBOutlet weak var lblMark: UILabel!
var Bean = Student()

//MARK:-ViewController LifeCycel
override func viewDidLoad() {
    super.viewDidLoad()
    print(Bean)
   self.lblEmail.text = Bean.student_Email
   self.lblName.text = Bean.student_Name
   self.lblMark.text = String(Bean.student_Mobile)
   self.lblcollage.text = Bean.student_Collage
   
}
//MARK:- uibutton Action
@IBAction func TapTOback(_ sender: Any) {
    self.navigationController!.popViewController(animated: true)
}
}










