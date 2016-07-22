# SignalRImports System.Web
Imports Microsoft.AspNet.SignalR
Imports System.Threading.Tasks
Imports System.Threading
Imports StackExchange.Redis

Namespace WorkOrderView
    Public Class WorkOrderViewHub
        Inherits Hub
       
        Public Property db As StackExchange.Redis.IDatabase

        Dim grpNamesKey As String
        Dim userObject As New MasterDataLibrary.User
        Dim userID As String


        Private Const RedisDBNum As Integer = 5
        Public Sub New()
            db = RedisCache.Connection.GetDatabase(RedisDBNum)

        End Sub

        Public Overrides Function OnConnected() As Task
            grpNamesKey = Context.QueryString("groupNo")
            userID = Context.User.Identity.Name.Split("|")(0)
            UpdateDictionary()
            UpdateSharedGroupNames()


            Groups.Add(Context.ConnectionId, grpNamesKey)
            BroadCastViewersAdd()
            Return MyBase.OnConnected()
        End Function

        Public Overrides Function OnDisconnected(stopCalled As Boolean) As Task

            grpNamesKey = Context.QueryString("groupNo")
            userID = Context.User.Identity.Name.Split("|")(0)
            Groups.Remove(Context.ConnectionId, grpNamesKey)
            Dim fstring = db.SetMembers(userID)(0).ToString()
            Dim memberCollectionKey = grpNamesKey & "|" & fstring


            If db.KeyExists(memberCollectionKey) Then
                db.SetRemove(memberCollectionKey, Context.ConnectionId)
                If db.SetLength(memberCollectionKey) = 0 Then
                    db.KeyDelete(memberCollectionKey)
                    db.SetRemove(grpNamesKey, fstring)
                End If
                If db.SetLength(grpNamesKey) = 0 Then
                    db.KeyDelete(grpNamesKey)
                End If
            End If
            Thread.Sleep(1000)
            If Not db.KeyExists(memberCollectionKey) Then
                Clients.OthersInGroup(grpNamesKey).broadcastMessageRemove(fstring)
            End If

            Return MyBase.OnDisconnected(stopCalled)
        End Function

        Public Sub BroadCastViewersAdd()
            Clients.Client(Context.ConnectionId).broadcastMessageAdd(db.SetMembers(grpNamesKey).Distinct())
        End Sub
        Public Sub BroadCastViewersRemove()
            Thread.Sleep(1000)
       
        End Sub

        Public Sub UpdateDictionary()
            If Not db.KeyExists(userID) Then
                userObject = UserManager.GetById(userID, MetaSites.NestPortal)
                db.SetAdd(userID, userID + "|" + userObject.Fullname.Trim())
            End If
        End Sub

        Public Sub UpdateSharedGroupNames()
            Dim FullString = db.SetMembers(userID)
            Dim memberCollectionKey = grpNamesKey & "|" & FullString(0).ToString()
            If Not db.KeyExists(memberCollectionKey) Then
                db.SetAdd(memberCollectionKey, Context.ConnectionId)
            Else
                db.SetAdd(memberCollectionKey, Context.ConnectionId)
            End If
            If Not db.KeyExists(grpNamesKey) Then
                db.SetAdd(grpNamesKey, FullString)
            Else
                If Not db.SetContains(grpNamesKey, FullString(0).ToString()) Then
                    db.SetAdd(grpNamesKey, FullString(0).ToString())
                    Clients.OthersInGroup(grpNamesKey).broadcastMessageAddUser(FullString(0).ToString())
                End If
            End If
        End Sub

        Public Sub RemoveValuesFromSharedDictionary()

        End Sub
    End Class
End Namespace
